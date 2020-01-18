# It's a date!

This challenge is all about weird and wacky date- and time formats. I solved
it mainly using the Date::Parse perl module, with a few tweaks here and
there.

Rather than trying to build perl for Fenix, I set up a port forward on the
host VM, and a netcat listener on Fenix, so I could interact with the binary
over the network.

```
root@sphinx:/# sysctl net.ipv4.conf.all.forwarding=1
root@sphinx:/# iptables -t nat -A PREROUTING  -i enp0s3 -p tcp --dport 1234 -j DNAT --to-destination 10.42.0.2
root@sphinx:/# iptables -t nat -A POSTROUTING -o fenix                      -j SNAT --to-source      10.42.0.1

johnny@sphinx # nc -l -p 1234 -e ./itsadate
```


I wrote the following perl script to talk to the challenge binary:

```perl
#!/usr/bin/perl

use strict;

use IO::Socket::INET;
use Date::Parse qw( strptime );
use POSIX qw( mktime strftime );

my $S = new IO::Socket::INET(PeerAddr => "10.0.0.90:1234") or die;

$ENV{'TZ'} = 'UTC'; # Ensures mktime operates in UTC

while (defined(my $line = $S->getline())) {
	print($line);
	if ($line =~ /Day \d+, .*: +(.*)$/) {
		my $t = $1;

		# Normalize a few things that are easily handled using
		# regular expressions, and that the date parser doesn't
		# recognize by itself.
		$t =~ s/'(..)/$1 < 40 ? 2000+$1 : 1900+$1/e;
		$t =~ s/^(\d{4}) (...[^ ]*) (\d\d) /$3 $2 $1 /;
		$t =~ s/^(\d+).. day of (\d{4}) /$2-01-$1 /;
		$t =~ s/ day (\d+) /-01-$1 /;
		$t =~ s/^(\d+).. of (...[^ ]*) (\d{4}) /$1 $2 $3 /;
		$t =~ s/^(\d\d)-(\d\d)-(\d\d) /sprintf("%4d-%02d-%02d ", $1 < 40 ? 2000+$1 : 1900+$1, $2, $3)/e;

		# Parse the date into its component parts.

		my @t;
		# Plain UTC timestamp.
		if ($t =~ /^((?:19|20)..)(..)(..)(..)(..)(..) \+0000$/) {
			@t = ($6,$5,$4, $3,$2-1,$1-1900);
		}
		# Treat plain integers as Unix timestamps.
		elsif ($t =~ /^\d+$/) {
			@t = gmtime($t);
		}
		# Otherwise, just let Date::Parse handle it.
		else {
			@t = strptime($t);

			# Adjust for timezones.
			if (@t[6] > 0) {
				$t[0] -= $t[6];
				$t[6] = 0;
				@t = gmtime(mktime(@t));
			}
		}

		# Map 2-digit years into a 1940-2039 range.
		if ($t[5] < 40) { $t[5] += 100 };

		# Normalize answer
		$t = strftime("%Y-%m-%d %H:%M:%S UTC", @t);

		print("\e[1mAnswer: $t\e[0m\n");
		$S->print("$t\r\n");
	}
}
```


## Example

A sample run looks like this:

```
Welcome, user!

Congratulations on your acceptance to the coveted internship program at
Itsadate Noperations, Department of Dating!

This is your time to shine! Our international dating services hook up couples
from all over the world. And *you* get to help bridge love across borders!

All new interns are, per regulations, subject to a mandatory two-day advanced
introductory course on timestamps and standards.

People use the strangest dating formats, and that simply will not do. No,
Itsadate Noperations' customers require easy to read timestamps in all things.
How else will they manage to meet their dates on time?

Y-m-d H:M:S UTC, pls.

Day 1,  1/50:    1827942949
Answer: 2027-12-04 17:55:49 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1,  2/50:    Thu, 28 Oct '71 00:13:39 +0000
Answer: 1971-10-28 00:13:39 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1,  3/50:    Tue, 21 Apr '09 06:33:37 +0000
Answer: 2009-04-21 06:33:37 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1,  4/50:    2035-09-26 15:19:52
Answer: 2035-09-26 15:19:52 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1,  5/50:    2034-08-13T02:34:13Z
Answer: 2034-08-13 02:34:13 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1,  6/50:    20260415014753 +0000
Answer: 2026-04-15 01:47:53 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1,  7/50:    2003-04-26T22:56:12Z
Answer: 2003-04-26 22:56:12 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1,  8/50:    Sat Oct 24 18:41:19 1981
Answer: 1981-10-24 18:41:19 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1,  9/50:    Sat, 04 Mar '06 02:23:59 +0000
Answer: 2006-03-04 02:23:59 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 10/50:    2031-10-31T20:19:44+0000
Answer: 2031-10-31 20:19:44 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 11/50:    2006-11-07T03:06:04Z
Answer: 2006-11-07 03:06:04 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 12/50:    Sun, 23 May 1999 12:45:42 +0000
Answer: 1999-05-23 12:45:42 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 13/50:    Fri Feb 27 05:27:12 2032
Answer: 2032-02-27 05:27:12 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 14/50:    Sat, 06 Mar '10 14:51:06 +0000
Answer: 2010-03-06 14:51:06 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 15/50:    19820317054559 +0000
Answer: 1982-03-17 05:45:59 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 16/50:    20340212063405 +0000
Answer: 2034-02-12 06:34:05 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 17/50:    Wed, 02 Jan '30 06:34:38 +0000
Answer: 2030-01-02 06:34:38 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 18/50:    20050512034839 +0000
Answer: 2005-05-12 03:48:39 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 19/50:    1970-10-13 00:08:43
Answer: 1970-10-13 00:08:43 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 20/50:    1985-01-04T03:17:05+0000
Answer: 1985-01-04 03:17:05 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 21/50:    2022-12-14 16:05:42
Answer: 2022-12-14 16:05:42 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 22/50:    Wed Aug 09 08:28:03 2023
Answer: 2023-08-09 08:28:03 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 23/50:    20200418T173901Z
Answer: 2020-04-18 17:39:01 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 24/50:    20170919140419 +0000
Answer: 2017-09-19 14:04:19 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 25/50:    20161126090822 +0000
Answer: 2016-11-26 09:08:22 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 26/50:    1970-11-01T19:15:29Z
Answer: 1970-11-01 19:15:29 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 27/50:    1380079067
Answer: 2013-09-25 03:17:47 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 28/50:    1997-08-08T03:56:08+0000
Answer: 1997-08-08 03:56:08 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 29/50:    2034-09-26T21:42:35Z
Answer: 2034-09-26 21:42:35 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 30/50:    Tue, 27 May 1997 04:17:17 +0000
Answer: 1997-05-27 04:17:17 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 31/50:    Mon, 03 Nov '14 13:46:27 +0000
Answer: 2014-11-03 13:46:27 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 32/50:    2028-09-20 02:34:40
Answer: 2028-09-20 02:34:40 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 33/50:    2018-09-21T22:15:47Z
Answer: 2018-09-21 22:15:47 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 34/50:    2017-07-25 17:12:11
Answer: 2017-07-25 17:12:11 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 35/50:    19960921T115903Z
Answer: 1996-09-21 11:59:03 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 36/50:    Sun, 11 Oct 1987 21:29:58 +0000
Answer: 1987-10-11 21:29:58 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 37/50:    2025-01-17T07:30:13+0000
Answer: 2025-01-17 07:30:13 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 38/50:    Fri, 07 Sep '18 20:58:09 +0000
Answer: 2018-09-07 20:58:09 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 39/50:    1292471651
Answer: 2010-12-16 03:54:11 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 40/50:    Thu, 16 Nov 2023 00:59:14 +0000
Answer: 2023-11-16 00:59:14 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 41/50:    20110320T063759Z
Answer: 2011-03-20 06:37:59 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 42/50:    Tue, 12 Nov '96 03:34:59 +0000
Answer: 1996-11-12 03:34:59 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 43/50:    Tue Oct 19 22:40:44 2032
Answer: 2032-10-19 22:40:44 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 44/50:    Sat, 30 May '20 00:38:47 +0000
Answer: 2020-05-30 00:38:47 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 45/50:    Wed, 04 Aug '82 19:46:34 +0000
Answer: 1982-08-04 19:46:34 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 46/50:    19930727102630 +0000
Answer: 1993-07-27 10:26:30 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 47/50:    Mon Feb 08 09:49:01 1971
Answer: 1971-02-08 09:49:01 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 48/50:    2010-07-21 16:44:18
Answer: 2010-07-21 16:44:18 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 49/50:    20330219013750 +0000
Answer: 2033-02-19 01:37:50 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 1, 50/50:    2028-05-02T20:45:31+0000
Answer: 2028-05-02 20:45:31 UTC
> Hey! It's a timestamp. I know this!
Correct!

FE{f1r57_t1m32_93t5_4_p4d_0n_7h3_h34d}

Well done! There's hope for you yet.
Only... That was the easy part.
Tick tock. The night is young.

Day 2,   1/100:    May 20 '79 11:33:27 UTC
Answer: 1979-05-20 11:33:27 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,   2/100:    2031-10-07T10:21:38+0000
Answer: 2031-10-07 10:21:38 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,   3/100:    Aug 29 '15 18:59:14 +02:30
Answer: 2015-08-29 16:29:14 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,   4/100:    Thu Jan 25 18:44:33 2007
Answer: 2007-01-25 18:44:33 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,   5/100:    '75 day 211 01:43:51 PM +17:30
Answer: 1975-07-29 20:13:51 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,   6/100:    320th day of 2030 05:35:47 AM +15:45
Answer: 2030-11-15 13:50:47 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,   7/100:    2013-05-04T15:44:14Z
Answer: 2013-05-04 15:44:14 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,   8/100:    Tue, 17 Mar 1981 03:36:55 +0000
Answer: 1981-03-17 03:36:55 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,   9/100:    2025 day 098 15:09:28 +21:45
Answer: 2025-04-07 17:24:28 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  10/100:    2010 day 295 05:58:08 +23:00
Answer: 2010-10-21 06:58:08 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  11/100:    288th day of 1974 08:00:21 PM +23:45
Answer: 1974-10-14 20:15:21 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  12/100:    06th of July 1998 07:49:19 UTC
Answer: 1998-07-06 07:49:19 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  13/100:    085th day of '98 04:04:29 AM +01:45
Answer: 1998-03-26 02:19:29 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  14/100:    18th of January '07 06:22:08 AM +05:30
Answer: 2007-01-18 00:52:08 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  15/100:    2028 day 231 19:16:45 +06:45
Answer: 2028-08-18 12:31:45 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  16/100:    1974 day 257 02:25:26 AM +16:15
Answer: 1974-09-13 10:10:26 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  17/100:    2020 February 28 07:24:56 +08:15
Answer: 2020-02-27 23:09:56 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  18/100:    147th day of '88 10:57:26 +05:15
Answer: 1988-05-26 05:42:26 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  19/100:    83-12-17 12:55:24 AM +19:45
Answer: 1983-12-16 05:10:24 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  20/100:    16th of April '14 06:42:22 +21:45
Answer: 2014-04-15 08:57:22 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  21/100:    109th day of '30 08:57:57 PM UTC
Answer: 2030-04-19 20:57:57 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  22/100:    1998-08-01 00:47:58
Answer: 1998-08-01 00:47:58 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  23/100:    Tue, 17 Mar '92 17:37:49 +0000
Answer: 1992-03-17 17:37:49 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  24/100:    Sun, 21 Aug '83 03:39:37 +0000
Answer: 1983-08-21 03:39:37 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  25/100:    84737831
Answer: 1972-09-07 18:17:11 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  26/100:    07/11/00 03:56:31 PM UTC
Answer: 2000-07-11 15:56:31 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  27/100:    '17 day 351 10:58:50 AM UTC
Answer: 2017-12-17 10:58:50 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  28/100:    25th of November 1984 16:45:55 +21:45
Answer: 1984-11-24 19:00:55 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  29/100:    2011-10-17 00:16:57
Answer: 2011-10-17 00:16:57 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  30/100:    '00 day 337 02:34:28 AM +17:45
Answer: 2000-12-01 08:49:28 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  31/100:    September 05 2018 08:30:16 PM +13:30
Answer: 2018-09-05 07:00:16 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  32/100:    1976 day 138 22:36:24 +01:00
Answer: 1976-05-17 21:36:24 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  33/100:    1767143247
Answer: 2025-12-31 01:07:27 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  34/100:    Mon, 14 Aug 2006 00:03:08 +0000
Answer: 2006-08-14 00:03:08 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  35/100:    2032 day 356 13:36:25 UTC
Answer: 2032-12-21 13:36:25 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  36/100:    185th day of 2017 08:15:10 +03:00
Answer: 2017-07-04 05:15:10 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  37/100:    November 15 2006 06:41:51 +19:15
Answer: 2006-11-14 11:26:51 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  38/100:    2008-08-08T20:30:22Z
Answer: 2008-08-08 20:30:22 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  39/100:    Dec 06 '30 11:42:03 AM +00:45
Answer: 2030-12-06 10:57:03 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  40/100:    December 10 1973 19:31:12 +21:45
Answer: 1973-12-09 21:46:12 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  41/100:    '80 Jan 24 04:01:04 +07:15
Answer: 1980-01-23 20:46:04 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  42/100:    Tue, 18 Dec '35 02:26:00 +0000
Answer: 2035-12-18 02:26:00 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  43/100:    March 31 2007 00:21:41 +17:30
Answer: 2007-03-30 06:51:41 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  44/100:    '00 day 109 02:23:35 AM +17:00
Answer: 2000-04-17 09:23:35 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  45/100:    Wed, 21 May 2036 02:23:06 +0000
Answer: 2036-05-21 02:23:06 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  46/100:    December 07 2020 19:57:51 UTC
Answer: 2020-12-07 19:57:51 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  47/100:    1975 November 24 07:04:49 UTC
Answer: 1975-11-24 07:04:49 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  48/100:    19710705005856 +0000
Answer: 1971-07-05 00:58:56 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  49/100:    12/12/79 19:48:56 +13:45
Answer: 1979-12-12 06:03:56 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  50/100:    2028-05-19T22:03:17+0000
Answer: 2028-05-19 22:03:17 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  51/100:    02/15/96 02:35:49 +12:45
Answer: 1996-02-14 13:50:49 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  52/100:    278th day of '29 18:27:33 +02:15
Answer: 2029-10-05 16:12:33 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  53/100:    January 12 2010 09:23:57 AM UTC
Answer: 2010-01-12 09:23:57 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  54/100:    28-08-29 11:52:23 PM +05:30
Answer: 2028-08-29 18:22:23 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  55/100:    24th of October '20 09:05:08 +22:00
Answer: 2020-10-23 11:05:08 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  56/100:    04th of February '20 04:24:04 +08:30
Answer: 2020-02-03 19:54:04 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  57/100:    21th of May '33 07:24:35 AM UTC
Answer: 2033-05-21 07:24:35 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  58/100:    1977-06-21 02:57:23
Answer: 1977-06-21 02:57:23 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  59/100:    26th of October 1983 18:51:35 +20:00
Answer: 1983-10-25 22:51:35 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  60/100:    20340704092302 +0000
Answer: 2034-07-04 09:23:02 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  61/100:    Jan 19 '78 12:57:24 +03:45
Answer: 1978-01-19 09:12:24 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  62/100:    1972 day 053 01:29:01 +04:15
Answer: 1972-02-21 21:14:01 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  63/100:    1994-07-16 10:36:25
Answer: 1994-07-16 10:36:25 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  64/100:    2037 day 014 07:27:12 +13:45
Answer: 2037-01-13 17:42:12 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  65/100:    May 22 2025 03:59:32 AM +07:45
Answer: 2025-05-21 20:14:32 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  66/100:    19990509T182837Z
Answer: 1999-05-09 18:28:37 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  67/100:    147th day of 1981 10:36:40 +05:45
Answer: 1981-05-27 04:51:40 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  68/100:    19750208T080911Z
Answer: 1975-02-08 08:09:11 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  69/100:    Feb 23 '16 08:39:21 +19:15
Answer: 2016-02-22 13:24:21 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  70/100:    016th day of '70 12:51:11 UTC
Answer: 1970-01-16 12:51:11 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  71/100:    1986-06-07T03:35:08Z
Answer: 1986-06-07 03:35:08 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  72/100:    20110322T151858Z
Answer: 2011-03-22 15:18:58 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  73/100:    19931022032150 +0000
Answer: 1993-10-22 03:21:50 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  74/100:    2035-02-08T04:20:27Z
Answer: 2035-02-08 04:20:27 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  75/100:    263th day of 2018 02:08:44 UTC
Answer: 2018-09-20 02:08:44 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  76/100:    2009-03-01T22:08:48+0000
Answer: 2009-03-01 22:08:48 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  77/100:    01th of May '75 18:02:18 +11:30
Answer: 1975-05-01 06:32:18 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  78/100:    04th of February '90 21:17:53 +00:00
Answer: 1990-02-04 21:17:53 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  79/100:    2037-05-09T05:03:23+0000
Answer: 2037-05-09 05:03:23 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  80/100:    241th day of '77 12:12:13 PM +07:30
Answer: 1977-08-29 04:42:13 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  81/100:    Mon, 11 Dec 2000 18:12:53 +0000
Answer: 2000-12-11 18:12:53 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  82/100:    Oct 17 '81 12:03:26 PM +04:00
Answer: 1981-10-17 08:03:26 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  83/100:    2035-06-17T00:32:59+0000
Answer: 2035-06-17 00:32:59 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  84/100:    July 01 1999 13:35:28 +19:45
Answer: 1999-06-30 17:50:28 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  85/100:    December 28 2010 09:35:51 AM +03:15
Answer: 2010-12-28 06:20:51 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  86/100:    1975 May 29 02:49:57 AM UTC
Answer: 1975-05-29 02:49:57 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  87/100:    1975-09-04T22:28:40Z
Answer: 1975-09-04 22:28:40 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  88/100:    32-08-01 15:18:34 +10:15
Answer: 2032-08-01 05:03:34 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  89/100:    1981 day 024 14:42:53 +01:00
Answer: 1981-01-24 13:42:53 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  90/100:    2026 June 25 11:28:00 +14:45
Answer: 2026-06-24 20:43:00 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  91/100:    2036-03-24 08:59:42
Answer: 2036-03-24 08:59:42 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  92/100:    102th day of 2022 03:32:10 +22:30
Answer: 2022-04-11 05:02:10 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  93/100:    1971-06-15 05:12:01 AM +10:00
Answer: 1971-06-14 19:12:01 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  94/100:    '31 day 001 02:27:25 PM +05:45
Answer: 2031-01-01 08:42:25 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  95/100:    28-04-23 08:34:15 AM +22:45
Answer: 2028-04-22 09:49:15 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  96/100:    94-09-26 10:00:32 +10:30
Answer: 1994-09-25 23:30:32 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  97/100:    Dec 30 '02 09:09:40 +03:00
Answer: 2002-12-30 06:09:40 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  98/100:    202th day of '21 01:37:21 AM +22:30
Answer: 2021-07-20 03:07:21 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2,  99/100:    2037-02-15 00:55:05
Answer: 2037-02-15 00:55:05 UTC
> Hey! It's a timestamp. I know this!
Correct!

Day 2, 100/100:    June 11 1973 02:59:44 +14:45
Answer: 1973-06-10 12:14:44 UTC
> Hey! It's a timestamp. I know this!
Correct!

FE{t1m3_h34l5_a11_w0und5}

Congratulations! It's a date!

You passed the introductory course, and have now started your full-time
unpaid internship. Want to get paid? Get yourself promoted to
Master Dater by answering this one bonus question. Interns will hate you.

Day 9001, BONUS QUESTION:    70-01-01 01:07:20 +19:30
Answer: 1969-12-31 05:37:20 UTC
> Hey! It's a timestamp. I know this!
Correct!

FE{th3_n19h7_i5_y0u2_0wn}

Wait a minute! How'd you get here? That question was unpossible!

PS... You're fired.
```


## Flags

FLAG 1: `FE{f1r57_t1m32_93t5_4_p4d_0n_7h3_h34d}`

FLAG 2: `FE{t1m3_h34l5_a11_w0und5}`

FLAG 3: `FE{th3_n19h7_i5_y0u2_0wn}`


---
_Peter Tirsek, 2020-01-20_
