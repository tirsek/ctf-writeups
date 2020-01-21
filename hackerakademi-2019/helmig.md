# Helmig

There are several ways to solve the Helmig challenge. I will present
the "regular" solution, plus what I believe are a couple of bugs, which
allow us to get both the regular and the elite flags with no effort at
all.

We're given a directory with write permissions, and a daemon process
maintains a set of FIFO pipes in that directory. Upon startup, there's
a readable `message` file, which contains the following information:

```
noob@sphinx # cat message
Knock knock!
Recover the secret sequence to get the flag.
```

After we read the message, the process creates a set of write-only
FIFOs, and the challenge is to find the correct sequence of writes to
these FIFOs to unlock access to the flag. Whenever we write to the
wrong file, the set of files is replaced with a message indicating
failure, and when we write to the correct file, that file is removed,
e.g.:

```
noob@sphinx # echo > 0
noob@sphinx # cat message
Wrong sequence!
noob@sphinx # ls
0  1  2  3  4  5  6  7  8  9
noob@sphinx # echo > 4
noob@sphinx # ls
0  1  2  3  5  6  7  8  9
```

If too much time elapses before the correct sequence is entered, the
system resets and scrambles the sequence, so the problem cannot be
solved by hand.


## The intended solution

Given the behavior described above, it's clear that we're meant to
construct an algorithm to locate the secret combination one digit at a
time. This can be done using a small shell script:

```sh
#!/bin/bash

cd /home/noob/helmig/challenge || exit

# Clear the message to start
cat message > /dev/null || exit

seq=''
redo=0

while true; do
	echo "Known sequence: <${seq:1}>"

	for i in *; do
		if [ $redo = 1 ]; then
			for s in $seq; do echo > $s; done
		fi

		echo -n "Try $i ... "
		echo > $i
		sleep 0.1

		if [ ! -e message ]; then
			echo "OK!"

			if [ -e flag ]; then
				cat flag
				exit
			fi

			seq="$seq $i"
			redo=0

			break
		else
			cat message
			redo=1
		fi
	done
done
```

A sample run looks like this:
```
noob@sphinx # ./solve_helmig.sh
Known sequence: <>
Try 0 ... Wrong sequence!
Try 1 ... Wrong sequence!
Try 2 ... Wrong sequence!
Try 3 ... OK!
Known sequence: <3>
Try 0 ... OK!
Known sequence: <3 0>
Try 1 ... OK!
Known sequence: <3 0 1>
Try 2 ... Wrong sequence!
Try 4 ... Wrong sequence!
Try 5 ... Wrong sequence!
Try 6 ... Wrong sequence!
Try 7 ... Wrong sequence!
Try 8 ... OK!
Known sequence: <3 0 1 8>
Try 2 ... Wrong sequence!
Try 4 ... Wrong sequence!
Try 5 ... Wrong sequence!
Try 6 ... Wrong sequence!
Try 7 ... OK!
Known sequence: <3 0 1 8 7>
Try 2 ... Wrong sequence!
Try 4 ... Wrong sequence!
Try 5 ... Wrong sequence!
Try 6 ... OK!
Known sequence: <3 0 1 8 7 6>
Try 2 ... Wrong sequence!
Try 4 ... Wrong sequence!
Try 5 ... OK!
Known sequence: <3 0 1 8 7 6 5>
Try 2 ... Wrong sequence!
Try 4 ... OK!
Known sequence: <3 0 1 8 7 6 5 4>
Try 2 ... Wrong sequence!
Try 9 ... OK!
Known sequence: <3 0 1 8 7 6 5 4 9>
Try 2 ... OK!
FE{det_er_mig_der_staar_herude_og_banker_paa}
```


## The elite solution (?)

After solving the challenge the "slow" way, the message file leaves
another hint for us:

```
noob@sphinx # cat message
Congrats!
One more flag is rewarded for an 31337 flawless run.
Hack harder!
Hint: events = POLLIN
```

At first, I thought the POLLIN hint meant that we're supposed to write
a program that opens all the files at once, then uses poll to determine
which file the daemon expects to read from next, but on second thought
that doesn't make a lot of sense. It's probably rather a hint that the
_daemon_ is the one doing polling of all the FIFOs, then checking for
readability in the correct sequence order, so as long as we can make
all the FIFOs readable at once, we will get a flawless solution.

There's no atomic way we can write to all the files at once, but it
appears that just doing it quickly is quite sufficient. I had a whole C
program ready to do it as fast as possible, but it seems we can simply
do it in a loop from the shell and get the same result:

```
noob@sphinx # ls
message
noob@sphinx # cat message > /dev/null
noob@sphinx # for a in $(seq 0 9); do echo > $a; done
noob@sphinx # ls
flag
noob@sphinx # cat flag
FE{d3T_Er_m1G_D3r_5tAAr_h3rUDe_0g_b4NkeR_pAA}
```

I'm really not sure if this is the intended elite solution or a bug,
because it seems like it takes no effort at all to do it this way.

If we introduce a tiny amount of delay between each write (in this case
an extra `echo` command), we end up with the normal flag instead:

```
noob@sphinx # ls
message
noob@sphinx # cat message > /dev/null
noob@sphinx # for a in $(seq 0 9); do echo > $a; echo > /dev/null; done
noob@sphinx # ls
flag
noob@sphinx # cat flag
FE{det_er_mig_der_staar_herude_og_banker_paa}
```

Introducing a slightly longer delay triggers the "wrong sequence" result:

```
noob@sphinx # ls
message
noob@sphinx # cat message > /dev/null
noob@sphinx # for a in $(seq 0 9); do echo > $a; sleep 0.01; done
noob@sphinx # ls
1  2  3  4  5  6  7  8  9  message
noob@sphinx # cat message
Wrong sequence!
```


## The "lay in front of the bulldozer" solution

One final no-effort solution I tripped over is this: Since the
directory is writable, we can pre-create the 0..9 files before reading
the message FIFO. This seems to trick the daemon into revealing the
flag as well, presumably because it ignores the errors when creating
the FIFOs, then successfully reads the plain files in the correct order
afterwards.

Some times this method yields the normal flag, other time it gives us
the elite flag.

```
noob@sphinx # cd helmig/challenge/
noob@sphinx # ls -l
total 0
prwxr--r-- 1 helmig root 0 Jan  1 00:00 message
noob@sphinx # touch {0..9}
noob@sphinx # ls -l
total 0
-rw-r--r-- 1 noob   user 0 Jan  1 00:00 0
-rw-r--r-- 1 noob   user 0 Jan  1 00:00 1
-rw-r--r-- 1 noob   user 0 Jan  1 00:00 2
-rw-r--r-- 1 noob   user 0 Jan  1 00:00 3
-rw-r--r-- 1 noob   user 0 Jan  1 00:00 4
-rw-r--r-- 1 noob   user 0 Jan  1 00:00 5
-rw-r--r-- 1 noob   user 0 Jan  1 00:00 6
-rw-r--r-- 1 noob   user 0 Jan  1 00:00 7
-rw-r--r-- 1 noob   user 0 Jan  1 00:00 8
-rw-r--r-- 1 noob   user 0 Jan  1 00:00 9
prwxr--r-- 1 helmig root 0 Jan  1 00:00 message
noob@sphinx # cat message > /dev/null
noob@sphinx # ls -l
total 0
prwxr--r-- 1 helmig root 0 Jan  1 00:00 flag
noob@sphinx # cat flag
FE{d3T_Er_m1G_D3r_5tAAr_h3rUDe_0g_b4NkeR_pAA}
```


## Flags

1. `FE{det_er_mig_der_staar_herude_og_banker_paa}`
2. `FE{d3T_Er_m1G_D3r_5tAAr_h3rUDe_0g_b4NkeR_pAA}`


---
_Peter Tirsek, 2020-01-20_
