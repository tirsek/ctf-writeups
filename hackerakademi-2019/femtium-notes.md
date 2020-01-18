# Notes on the Femtium architecture

Femtium — or FEmtium? :) — is an experimental and fictitious RISC CPU
architecture developed by "Forsvarets Efterretningstjeneste", the
Danish Defense Intelligence Service (FE-DDIS) for educational purposes.

This document is a work in progress and may contain errors and omissions.


## Specifications

* 64 32-bit registers
* Most registers are general-purpose, although the ABI assigns certain
  registers specific meaning.
* 32-bit virtual address space
* Some (?) memory management features


## History

As far as I can tell, the first time the public saw a hint of the Femtium
architecture was in 2017 with the release of the FE-DDIS "hacker academy"
[recruiting challenge](https://fe-ddis.dk/SiteCollectionImages/FE/grafik_og_billeder/3zone-billeder/hacker-opgave2017.bmp).

The challenge showed a partial implementation of a Femtium emulator written
in 32-bit x86 assembly language, along with a base-64 encoded version of a
disk image for the Femtium to run. The challenge at that time was to fully
implement a working emulator, and run the code found on that disk image.

Without a lot of information to go on in 2017, I had made lots of
assumptions about the underlying architecture, and seeing the system again
in this year's CTF brought back fond memories along with some recognition of
the ABI and register assignments. Seeing an authoritative and fully
implemented emulator helped confirm many of the guesses I made originally.


## Registers

```
 #  Name Notes
 0  zz   Zero register; hardwired to always read 0

 1  v0
 2  v1

 3  a0   Used to pass function arguments and return values
 4  a1
 5  a2
 6  a3
 7  a4
 8  a5
 9  a6

10  s0   Callee-saved general purpose
11  s1
12  s2
... ...
27  s17
28  s18
29  s19

30  t0   Caller-saved general purpose
31  t1
32  t2
... ...
49  t19
50  t20
51  t21

52  x
53  y
54  z
55  k0
56  k1
57  at0  Address Temporary
58  at1  Address Temporary
59  ra   Return Address
60  fp   Frame Pointer
61  gp
62  sp   Stack Pointer
63  ip   Instruction Pointer / Program Counter
```


## Instruction set

For more information, see the Femtium Instruction Reference included with
the CTF itself.


---
_Peter Tirsek, 2020-01-20_
