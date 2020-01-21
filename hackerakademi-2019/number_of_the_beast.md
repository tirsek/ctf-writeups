# Number of the beast

We're given a text file with what at first glance appears to be a hex-encoded
flag, intermixed with lots of gibberish.

FE{feff00٦۹00৫f00௬൰0๖d00٥f00৬300௬f00๗໕00٦e00৭੪00௬౯00๖e00٦۷00৫f00௬f00๖e00٥f00৭੯00௬f00๗໕00٥f00৭੪00௬౮00๖໙00٧300৫f00௭౪00๖໙00٦d00৬੫}

Of course it doesn't help that my go-to console font has terrible glyphs for
many Unicode characters. Looking at the hex dump just made things look even
more confusing:

```
johnny@sphinx # hexdump -C number_of_the_beast.txt
00000000  46 45 7b 66 65 66 66 30  30 d9 a6 db b9 30 30 e0  |FE{feff00....00.|
00000010  a7 ab 66 30 30 e0 af ac  e0 b5 b0 30 e0 b9 96 64  |..f00......0...d|
00000020  30 30 d9 a5 66 30 30 e0  a7 ac 33 30 30 e0 af ac  |00..f00...300...|
00000030  66 30 30 e0 b9 97 e0 bb  95 30 30 d9 a6 65 30 30  |f00......00..e00|
00000040  e0 a7 ad e0 a9 aa 30 30  e0 af ac e0 b1 af 30 30  |......00......00|
00000050  e0 b9 96 65 30 30 d9 a6  db b7 30 30 e0 a7 ab 66  |...e00....00...f|
00000060  30 30 e0 af ac 66 30 30  e0 b9 96 65 30 30 d9 a5  |00...f00...e00..|
00000070  66 30 30 e0 a7 ad e0 a9  af 30 30 e0 af ac 66 30  |f00......00...f0|
00000080  30 e0 b9 97 e0 bb 95 30  30 d9 a5 66 30 30 e0 a7  |0......00..f00..|
00000090  ad e0 a9 aa 30 30 e0 af  ac e0 b1 ae 30 30 e0 b9  |....00......00..|
000000a0  96 e0 bb 99 30 30 d9 a7  33 30 30 e0 a7 ab 66 30  |....00..300...f0|
000000b0  30 e0 af ad e0 b1 aa 30  30 e0 b9 96 e0 bb 99 30  |0......00......0|
000000c0  30 d9 a6 64 30 30 e0 a7  ac e0 a9 ab 7d           |0..d00......}|
000000cd
```

When I finally realized that this is indeed real Unicode characters and not
some encrypted, compressed, or decoy data, everything came together pretty
quickly.

The characters are 16-bit hexadecimal representations of Unicode characters,
starting with "feff", the ZERO WIDTH NO-BREAK SPACE that is usually used as a
Unicode byte-order mark.

The strange looking characters turn out all to be numerals from different
alphabets. I looked some of them up in full, but for the rest of them, Googling
the non-ASCII numerals one by one was very helpful.

|0|1|2|3|4|5|6|7|8|9| Unicode range   | Description           |
|-|-|-|-|-|-|-|-|-|-|-----------------|-----------------------|
|٠|١|٢|٣|٤|٥|٦|٧|٨|٩| U+0660 – U+0669 | Arabic-Indic numerals |
|۰|۱|۲|۳|۴|۵|۶|۷|۸|۹| U+06F0 – U+06F9 | Extended Arabic-Indic |
|০|১|২|৩|৪|৫|৬|৭|৮|৯| U+09E6 – U+09EF | Bengali               |
|௦|௧|௨|௩|௪|௫|௬|௭|௮|௯| U+0BE6 – U+0BEF | Tamil                 |
|๐|๑|๒|๓|๔|๕|๖|๗|๘|๙| U+0E50 – U+0E59 | Thai                  |
|໐|໑|໒|໓|໔|໕|໖|໗|໘|໙| U+0ED0 – U+0ED9 | Lao                   |

The one curveball here is the '൰' character, MALAYALAM NUMBER TEN (U+0D70).
When translating back to ASCII, this character is replaced by TWO digits, '1'
and '0'.

The full translation is as follows:

| Encoded | Decoded | ASCII | Notes                                          |
|---------|---------|-------|------------------------------------------------|
| 00٦۹    | 0069    | i     |                                                |
| 00৫f    | 005f    | _     |                                                |
| 00௬൰    | 0061(0) | a     | (The extra '0' is used for the next character) |
|  0๖d    | 006d    | m     |                                                |
| 00٥f    | 005f    | _     |                                                |
| 00৬3    | 0063    | c     |                                                |
| 00௬f    | 006f    | o     |                                                |
| 00๗໕    | 0075    | u     |                                                |
| 00٦e    | 006e    | n     |                                                |
| 00৭੪    | 0074    | t     |                                                |
| 00௬౯    | 0069    | i     |                                                |
| 00๖e    | 006e    | n     |                                                |
| 00٦۷    | 0067    | g     |                                                |
| 00৫f    | 005f    | _     |                                                |
| 00௬f    | 006f    | o     |                                                |
| 00๖e    | 006e    | n     |                                                |
| 00٥f    | 005f    | _     |                                                |
| 00৭੯    | 0079    | y     |                                                |
| 00௬f    | 006f    | o     |                                                |
| 00๗໕    | 0075    | u     |                                                |
| 00٥f    | 005f    | _     |                                                |
| 00৭੪    | 0074    | t     |                                                |
| 00௬౮    | 0068    | h     |                                                |
| 00๖໙    | 0069    | i     |                                                |
| 00٧3    | 0073    | s     |                                                |
| 00৫f    | 005f    | _     |                                                |
| 00௭౪    | 0074    | t     |                                                |
| 00๖໙    | 0069    | i     |                                                |
| 00٦d    | 006d    | m     |                                                |
| 00৬੫    | 0065    | e     |                                                |


## Flag

`FE{i_am_counting_on_you_this_time}`


---
_Peter Tirsek, 2020-01-20_
