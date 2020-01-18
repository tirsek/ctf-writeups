# Gaining root on the Fenix OS

This wasn't an official challenge, but after I completed all the other
challenges, I decided I wanted to gain root access on the Fenix machine
in order to see if, by chance, any extra bonus flag was hidden that
didn't show up in the official list of challenges. While that wasn't
the case, finding a way to gain root was still an interesting problem
to solve.


## Analyzing the emulator

Since we have access to the entire Fenix machine, the easiest way in is
probably to simply replace the hashed root password in `/etc/shadow`
with a new one. In order to do that, we need access to the disk image.
The disk image is encrypted, but since either the emulator or the Fenix
kernel is able to access the encrypted image, it must be possible to
reverse this process and access the underlying raw data somehow.

Fortunately, the emulator binary is chock full of debug information. We
don't have its source code, but the binary has symbol and line number
information included, which makes running it under gdb much easier. To
find the function that accesses the disk image, I started by stepping
through the main function to find where the name of the disk image
(given on the command line) is stored. After a little digging, I found
a global `struct kernel` variable named `fenix` with a field named
`arch.args.bootdisk_image`, which gets populated inside the
`parse_args` function.

Setting a read watch on that variable led me to the `dsk_init` function
where the file is opened, and its file descriptor is stored in
`fenix.bootdisk.arch.fd`. Another read watch on that location leads to
the read function, aptly named `dsk_read_sector`. That function calls
pread to read the desired sector number from the disk image, and if the
read is requested from bus_id 20, which indicates the bootdisk_image,
the function proceeds to decrypt the sector contents.

The decryption starts by initializing an array of 16 uint32_t's with
the characters "fenixdiskcrypty", not as a string, but as if they're
32-bit wide characters. The last int of the array is initialized with
the sector number being read. After that, the function executes a lot
of shifts, rotates, and xor operations. A bit of further digging along
with some references in the emulator's debug information turns out that
this is the salsa20 cipher/hash function. After this "fenixdiskcrypty"
and the sector number is scrambled up, the emulator XORs the sector
contents with the values from this hash to get the plaintext sector
data.


## Fenix disk image encryption

Based on the information discovered in the analysis, I wrote a small
program to decrypt the Fenix disk image. The encryption is completely
symmetrical, so the same tool can be used to re-encrypt the modified
disk image as well.

```c
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

/* From https://cr.yp.to/salsa20.html */
#define R(a,b) (((a) << (b)) | ((a) >> (32 - (b))))
void salsa20_word_specification(uint32_t out[16],uint32_t in[16])
{
  int i;
  uint32_t x[16];
  for (i = 0;i < 16;++i) x[i] = in[i];
  for (i = 20;i > 0;i -= 2) {
    x[ 4] ^= R(x[ 0]+x[12], 7);  x[ 8] ^= R(x[ 4]+x[ 0], 9);
    x[12] ^= R(x[ 8]+x[ 4],13);  x[ 0] ^= R(x[12]+x[ 8],18);
    x[ 9] ^= R(x[ 5]+x[ 1], 7);  x[13] ^= R(x[ 9]+x[ 5], 9);
    x[ 1] ^= R(x[13]+x[ 9],13);  x[ 5] ^= R(x[ 1]+x[13],18);
    x[14] ^= R(x[10]+x[ 6], 7);  x[ 2] ^= R(x[14]+x[10], 9);
    x[ 6] ^= R(x[ 2]+x[14],13);  x[10] ^= R(x[ 6]+x[ 2],18);
    x[ 3] ^= R(x[15]+x[11], 7);  x[ 7] ^= R(x[ 3]+x[15], 9);
    x[11] ^= R(x[ 7]+x[ 3],13);  x[15] ^= R(x[11]+x[ 7],18);
    x[ 1] ^= R(x[ 0]+x[ 3], 7);  x[ 2] ^= R(x[ 1]+x[ 0], 9);
    x[ 3] ^= R(x[ 2]+x[ 1],13);  x[ 0] ^= R(x[ 3]+x[ 2],18);
    x[ 6] ^= R(x[ 5]+x[ 4], 7);  x[ 7] ^= R(x[ 6]+x[ 5], 9);
    x[ 4] ^= R(x[ 7]+x[ 6],13);  x[ 5] ^= R(x[ 4]+x[ 7],18);
    x[11] ^= R(x[10]+x[ 9], 7);  x[ 8] ^= R(x[11]+x[10], 9);
    x[ 9] ^= R(x[ 8]+x[11],13);  x[10] ^= R(x[ 9]+x[ 8],18);
    x[12] ^= R(x[15]+x[14], 7);  x[13] ^= R(x[12]+x[15], 9);
    x[14] ^= R(x[13]+x[12],13);  x[15] ^= R(x[14]+x[13],18);
  }
  for (i = 0;i < 16;++i) out[i] = x[i] + in[i];
}

static uint32_t key[16] = {'f','e','n','i','x','d','i','s','k','c','r','y','p','t','y'};

int main()
{
	union {
		uint8_t u8[512];
		uint32_t u32[512/4];
	} sector;

	for (key[15]=0; key[15] < 1638416; key[15]++) {
		uint32_t xor[16];
		salsa20_word_specification(xor, key);

		int n = read(0, sector.u8, 512);
		if (n < 512) { perror("read"); exit(1); }

		for (int i=0; i<512/4; i++) {
			sector.u32[i] ^= xor[i & 15];
		}

		write(1, sector.u8, 512);
	}
}
```


## Replacing the root password

Now that we can decrypt the Fenix boot image, we can find the location
of the root password within the `/etc/shadow` file, replace it, and
reencrypt the boot image:

```
user@sphinx:~$ ./encrypt-fenix-img < fenix.img > decrypted-fenix.img
user@sphinx:~$ strings -t d decrypted-fenix.img | grep 'root:\$1\$'
6701056 root:$1$Cm6PS1fl$lHosWpriq1F/ENYnifGc0.:17873:0:99999:7:::
user@sphinx:~$ echo root | mkpasswd --method=md5 --stdin
$1$heGwEj.6$M1f54jpaN1hdoLSm7n.JH.
user@sphinx:~$ echo -n '$1$heGwEj.6$M1f54jpaN1hdoLSm7n.JH.' | dd of=decrypted-fenix.img conv=notrunc seek=6701061 bs=34 count=1 oflag=seek_bytes
user@sphinx:~$ ./encrypt-fenix-img  < decrypted-fenix.img > new-fenix.img
```

Finally, we can boot up Fenix again, and log in with root/root:

```
user@sphinx:~$ ./emulator new-fenix.img
GURB
=D= Kernel found: fnode 10 (size 556480)
=D= ELF load [entry 0x800020f4]
=*= FENIX [NATIVE]
[...]

*
| sphinx::/dev/tty1
|
| fenix 1.3.37 online
*
sphinx login: root
Password:
login[6]: root login on 'tty1'
root@sphinx # id
uid=0(root) gid=0(root) groups=0(root)
```


---
_Peter Tirsek, 2020-01-20_
