# Binary exploit 1

We're given a [Femtium](femtium-notes.md) binary, that when run asks for a
password. The flag is the password, so we need to analyze the code of this
binary to find the password.

The binary is very simple. It is written in a very linear fashion, it
doesn't appear to be linked against libc, and it uses syscalls directly.

First it copies the password prompt, "Password please: ", onto the stack one
character at a time, and makes a syscall to write it out

```
00401000: 47df00ef  add     sp, sp,-17          ; Allocate local variable space
00401004: 806000a4  movi    a0, 0x00000050      ; 'P'; 80
00401008: 207f0000  stb     a0, [sp+0]
0040100c: 80600c20  movi    a0, 0x00000061      ; 'a'; 97
00401010: 207f0001  stb     a0, [sp+1]
00401014: 80600e60  movi    a0, 0x00000073      ; 's'; 115
00401018: 207f0002  stb     a0, [sp+2]
...
00401074: 80600ca0  movi    a0, 0x00000065      ; 'e'; 101
00401078: 207f000e  stb     a0, [sp+14]
0040107c: 806003a1  movi    a0, 0x0000003a      ; ':'; 58
00401080: 207f000f  stb     a0, [sp+15]
00401084: 80600025  movi    a0, 0x00000020      ; ' '; 32
00401088: 207f0010  stb     a0, [sp+16]
0040108c: 80607d22  movi    a0, 0x00000fa4      ; 4004
00401090: 80800000  movi    a1, 0x00000000      ; 0
00401094: 80a00000  movi    a2, 0x00000000      ; 0
00401098: 40bf0000  mov     a2, sp
0040109c: 80c00220  movi    a3, 0x00000011      ; 17
004010a0: e8000000  sys
004010a4: 47df0011  add     sp, sp,17           ; Free local variable space
```

Then, it allocated 24 bytes on the stack for the input buffer and stores the
address of that in registers `s2` and `s4`, and in an unrolled loop, it
reads 24 characters from the input one at a time into that buffer:

```
004010a8: 47df00e8  add     sp, sp,-24          ; Allocate local variable space
004010ac: 81800000  movi    s2, 0x00000000      ; 0
004010b0: 419f0000  mov     s2, sp
004010b4: 81c00000  movi    s4, 0x00000000      ; 0
004010b8: 41df0000  mov     s4, sp

004010bc: 8061f460  movi    a0, 0x00000fa3      ; 4003
004010c0: 80800000  movi    a1, 0x00000000      ; 0
004010c4: 80a00000  movi    a2, 0x00000000      ; 0
004010c8: 40bf0000  mov     a2, sp
004010cc: 80c00020  movi    a3, 0x00000001      ; 1
004010d0: e8000000  sys

004010d4: 8061f460  movi    a0, 0x00000fa3      ; 4003
004010d8: 80800000  movi    a1, 0x00000000      ; 0
004010dc: 80a00000  movi    a2, 0x00000000      ; 0
004010e0: 40bf0001  add     a2, sp,1
004010e4: 80c00020  movi    a3, 0x00000001      ; 1
004010e8: e8000000  sys

...

004012e4: 8061f460  movi    a0, 0x00000fa3      ; 4003
004012e8: 80800000  movi    a1, 0x00000000      ; 0
004012ec: 80a00000  movi    a2, 0x00000000      ; 0
004012f0: 40bf0017  add     a2, sp,23
004012f4: 80c00020  movi    a3, 0x00000001      ; 1
004012f8: e8000000  sys
```

After that, two 24-byte arrays are initialized on the stack, containing the
data that gets combined to check the password. The address of the first
array is stored in register `s0`, and the second in `s1`. We'll call the
first the "answer" array, and the other the "secret" array.

```
004012fc: 47df00e8  add     sp, sp,-24          ; Allocate local variable space
00401300: 81400000  movi    s0, 0x00000000
00401304: 415f0000  mov     s0, sp              ; Store pointer in s0
00401308: 80600de0  movi    a0, 0x0000006f      ; 'o'; 111
0040130c: 207f0000  stb     a0, [sp+0]
00401310: 80600ce0  movi    a0, 0x00000067      ; 'g'; 103
00401314: 207f0001  stb     a0, [sp+1]
...
004013b8: 80600ea1  movi    a0, 0x000000ea      ; 234
004013bc: 207f0016  stb     a0, [sp+22]
004013c0: 80600aa1  movi    a0, 0x000000aa      ; 170
004013c4: 207f0017  stb     a0, [sp+23]

004013c8: 47df00e8  add     sp, sp,-24          ; Allocate local variable space
004013cc: 81600000  movi    s1, 0x00000000
004013d0: 417f0000  mov     s1, sp              ; Store pointer in s1
004013d4: 80600063  movi    a0, 0x00000018      ; 24
004013d8: 207f0000  stb     a0, [sp+0]
004013dc: 80600220  movi    a0, 0x00000011      ; 17
004013e0: 207f0001  stb     a0, [sp+1]
...
0040148c: 806000a0  movi    a0, 0x00000005      ; 5
00401490: 207f0017  stb     a0, [sp+23]
```

Eventually the two arrays contain the following data:

```
s0: 6f 67 f0 79 e0 e4 b2 ae  6d 69 65 6c 63 68 61 69  8a 92 a2 b0 b4 ef ea aa
s1: 18 11 64 0f 60 5e 42 32  03 06 12 33 25 0d 0c 1d  07 01 17 31 24 9b 4d 05
```

Next, another unrolled loop checks each of the 24 input bytes for LF or NUL
characters, and jumps to the label _fail if any match is found. This is done
to ensure the input string is of the correct length (or at least long
enough).


## The password verification

The main part of the binary is divided into 3 parts, each checking 8 bytes
of the password.

The first 8 bytes is combined with the first 8 bytes of the secret array
like this:

```
00401674: 00660000  ldb     a0, [s2+0]          ; Load the next byte of input
00401678: 00858000  ldb     a1, [s1+0]          ; Load the next byte of secret
0040167c: 40818800  add     a1, a0,a1,0         ; Add them together
00401680: 40820011  add     a1, a1,17           ; Add 17
00401684: 20858000  stb     a1, [s1+0]          ; Store back to secret
00401688: 41860001  add     s2, s2,1            ; Increment input pointer
0040168c: 41658001  add     s1, s1,1            ; Increment secret pointer
```

This will eventually be compared to the answers, so in essence, for the
first 8 bytes, input + secret + 17 must match the answer byte, which means
we can reverse this as "input = answer - secret - 17".

```
    Answer: 6f 67 f0 79 e0 e4 b2 ae
  - Secret: 18 11 64 0f 60 5e 42 32
  - 17:     11 11 11 11 11 11 11 11
  = Input:  46 45 7b 59 6f 75 5f 6b  "FE{You_k"
```


The next 8 bytes are simply XOR'ed with the bytes from the secret, then
stored back in the secret:

```
00401754: 00660000  ldb     a0, [s2+0]          ; Load the next byte of input
00401758: 00858000  ldb     a1, [s1+0]          ; Load the next byte of secret
0040175c: 5f418800  nor     at1, a0,a1,0        ; Long-form XOR
00401760: 587d0600  nor     a0, at1,a0,0
00401764: 5f3d0800  nor     at0, at1,a1,0
00401768: 59a1f200  nor     s3, a0,at0,0
0040176c: 59a69a00  not     s3
00401770: 21a58000  stb     s3, [s1+0]          ; Store back to secret
00401774: 41860001  add     s2, s2,1            ; Increment input pointer
00401778: 41658001  add     s1, s1,1            ; Increment secret pointer
```

The long-form XOR is interesting. Even though the updated Femtium CPU has an
instruction that is natively capable of performing an XOR operation, the
original Femtium implementation from 2017 only had a NOR operation. XOR can
be implemented using only NOR instructions and a few temporaries, and that's
how it was done in the original challenge code in 2017 as well.

```
    |  step1  |    step2    |    step3    |      step4      |   step5   |
a b | a NOR b | step1 NOR a | step1 NOR b | step2 NOR step3 | NOT step4 |
----|---------|-------------|-------------|-----------------|-----------|
0 0 |    1    |      0      |      0      |        1        |     0     |
0 1 |    0    |      1      |      0      |        0        |     1     |
1 0 |    0    |      0      |      1      |        0        |     1     |
1 1 |    0    |      0      |      0      |        1        |     0     |
```

As before, these 8 bytes will be compared to the answer array, so to recover
this part of the password, we'll simply have to XOR the answer with the
secret:

```
    Answer: 6d 69 65 6c 63 68 61 69
XOR Secret: 03 06 12 33 25 0d 0c 1d
  = Input:  6e 6f 77 5f 46 65 6d 74  "now_Femt"
```

The final 8 bytes are added one by one to the last 8 bytes of the secret,
plus a constant. Unlike the first 8 bytes, however, the extra term of the
addition is incremented by 2 for each byte.

```
00401894: 00660000  ldb     a0, [s2+0]          ; Load the next byte of input
00401898: 00858000  ldb     a1, [s1+0]          ; Load the next byte of secret
0040189c: 4081881a  add     a1, a0,a1,26        ; Add them together, plus 26
004018a0: 40820000  add     a1, a1,0            ; Add an extra 0
004018a4: 20858000  stb     a1, [s1+0]          ; Store back to secret
004018a8: 41860001  add     s2, s2,1            ; Increment input pointer
004018ac: 41658001  add     s1, s1,1            ; Increment secret pointer

004018b0: 00660000  ldb     a0, [s2+0]          ; Next byte of input
004018b4: 00858000  ldb     a1, [s1+0]          ; Secret
004018b8: 4081881b  add     a1, a0,a1,27        ; Add them together, plus 27
004018bc: 40820001  add     a1, a1,1            ; Add an extra 1
004018c0: 20858000  stb     a1, [s1+0]          ; Store
004018c4: 41860001  add     s2, s2,1            ; Increment
004018c8: 41658001  add     s1, s1,1            ; Increment

...
```

Reversing this is like the first 8 bytes, only with a different offset:

```
    Answer: 8a 92 a2 b0 b4 ef ea aa
  - Secret: 07 01 17 31 24 9b 4d 05
  - Offset: 1a 1c 1e 20 22 24 26 28
  = Input:  69 75 6d 5f 6e 30 77 7d  "ium_n0w}"
```

Finally, an unrolled loop compares the now modified secret to the answer. If
the input was the correct flag, the secret will now be identical to the
answer, the compare will succeed, and the success message, "You did it!",
will be printed.


## Flag

`FE{You_know_Femtium_n0w}`


---
_Peter Tirsek, 2020-01-20_
