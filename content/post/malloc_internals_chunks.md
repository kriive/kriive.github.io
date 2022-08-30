---
title: "Malloc internals: chunks"
date: 2022-08-30T12:45:11+02:00
draft: true
---
Since last year I'm involved in infosec, and I co-founded the
[havce](https://havce.it) CTF team with some of the colleagues that attended
the CyberChallenge.IT course in 2021. 

I am a big fan of `pwn` challenges and binary exploitation in general,
so after dealing with standard buffer overflows on the stack and the various
format string vulnerabilities, I wanted to step up my skills and learn some
heap exploitation.

I started by using the awesome 
[HeapLAB](https://www.udemy.com/course/linux-heap-exploitation-part-1) by 
Max Kamper. If you're serious about heap exploitation, this course is gold, 
go and buy it. Max Kamper is also the author of the famous 
[**ROP emporium**](https://ropemporium.com/), super useful to learn return oriented programming.

In this post I will cover some basics about malloc internal structures 
to help me remember and understand better the attacks.

## What is `malloc`?
`malloc` is the name of one of the functions that the standard C library
provides to manually manage dynamic memory allocations.
It is also the name given to the glibc memory allocator. 

## Chunks
**chunks** are the fundamental unit of work in `malloc`. They usually are
part of the heap, but they can be allocated as separate entities with a 
call to `mmap`.

This is what a in-use chunk looks like.
```
                        ┌──────┐
malloc ptr->            │ size │
            ┌───────────┴──────┤
programptr->│     user_data    │
            │           ┌──────┘
            │           │
            └───────────┘
```
While programs usually deal with the programptr (a pointer to *user data*),
`malloc` usually deals with the malloc ptr (tcache is an exception). 
The malloc ptr begins **8 bytes before** the size field.

The `size` field of the chunk indicates the length of the user data plus 
the size field length. Since all chunks are multiples of 8 bytes, the 3
least significant bits (LSBs) of the chunk size field can be used for flags.

These flags are (listed from least significant to most significant):
 - `PREV_INUSE`: when set, the previous chunk is still being used by the
 application, when cleared the previous chunk is free'd.
 - `IS_MMAPPED`, when set, this chunk has been allocated with `mmap` and is 
 not part of a heap at all.
 - `NON_MAIN_ARENA`: when set, it indicates that the chunk does not belong
 to the main arena.

 When a chunk is free, up to 5 quadwords of its user data can be repurposed
 as `malloc` metadata and they can become part of the succeeding chunk.

 ``` C
/*
               ┌─────────────┐
   prev_size   │    size     │
┌─────────────┼─────────────┤
│ fd          │ bk          │
├─────────────┼─────────────┤
│ fd_nextsize │ bk_nextsize │
├─────────────┼─────────────┤
│             │             │
├─────────────┼─────────────┘
│             │             
└─────────────┘
*/

struct malloc_chunk {
  INTERNAL_SIZE_T      mchunk_prev_size;  /* Size of previous chunk (if free).  */
  INTERNAL_SIZE_T      mchunk_size;       /* Size in bytes, including overhead. */

  struct malloc_chunk* fd;         /* double links -- used only if free. */
  struct malloc_chunk* bk;

  /* Only used for large blocks: pointer to next larger size.  */
  struct malloc_chunk* fd_nextsize; /* double links -- used only if free. */
  struct malloc_chunk* bk_nextsize;
};

```
When a chunk is free, the same memory that was used by the application 
data before gets repurposed to store `malloc` internal metadata.

This is probably the main reason why the glibc memory allocator is such a
yummy target to exploit designers, as it's really easy to tamper with
the `malloc` metadata and use it as an exploit vector.

When a chunk is free, its *shape* can change. Like we said, some fields
can be repurposed, but some memory can actually change ownership, and 
*migrate* to another chunk.

In the following lines we're going to mention the **malloc bins**,
which are some lists used by `malloc` to keep track of the free'd chunks. 

There are many of them and they serve different purposes, but they are
basically linked lists of free chunks. Some of them are singly linked lists
and other are doubly linked lists. We will cover them in detail in 
future articles.

When a chunk is free'd, the first quadword of a chunk's user data gets
repurposed as a **forward pointer** (*fd*).

When the chunk gets linked in a bin that behaves like a doubly linked list
(e.g. the unsortedbin), the second quadword gets repurposed as a
**backwards pointer** (*bk*).

The third and fourth quadwords are repurposed as `fd_nextsize` and 
`bk_nextsize` in largebins only.

In bins that support consolidation, the last quadword of a free chunk gets
repurposed as the `prev_size` field. This indicates the size of the free
chunk, exactly like the `size` field does, but without the flags.

`malloc` considers the prev_size field as part of the succeeding chunk
(that's why I said chunks can change shape) and when it is set, the
`PREV_INUSE` flag of the succeeding chunk is also set.

On GLIBC versions >= 2.29, the second quadword of a free chunk linked
in a `tcachebin` gets repurposed as a `key` field, used to detect 
double-frees.

### Further reading
 - [Malloc internals](https://sourceware.org/glibc/wiki/MallocInternals),
covers everything we've covered and some.
 - [GLIBC malloc source code](https://elixir.bootlin.com/glibc/glibc-2.36/source/malloc/malloc.c), this is the actual source of truth of everything we 
 say.
