# Sort Sol

We're given a console-based Guitar-Hero-style game with 7 inputs, scrolling
along the screen:

```
1 | ---------o---------o---------o---------|---------|---------|---------|------
2 | ---------|---------|---------o---------o---------o---------o---------o------
3 | ---------|---------|---------o---------o---------o---------o---------o------
4 | ---------|---------|---------o---------|---------|---------|---------|------
5 | ---------o---------o---------|---------|---------|---------|---------o------
6 | ---------o---------|---------o---------|---------o---------o---------o------
7 | ---------|---------o---------o---------o---------o---------o---------o------
health: 3
```

The leftmost veritcal line in the screen's third column shows the currently
activated inputs. Entering the number of a string toggles the state of that
string in the input, and before the next active note hits the left side of
the screen, the input must be set accordingly, or we lose a health point for
each string that is not set correctly.

Each "note" contains one ASCII-encoded character of the flag, with string
"1" being the most significant bit, and 'o' / '|' indicating a one and a
zero bit, respectively:

```
1 |          1         1         1         0         0         0         0
2 |          0         0         1         1         1         1         1
3 |          0         0         1         1         1         1         1
4 |          0         0         1         0         0         0         0
5 |          1         1         0         0         0         0         1
6 |          1         0         1         0         1         1         1
7 |          0         1         1         1         1         1         1

             F         E         {         1         3         3         7
```

The game scrolls too fast to be played by hand, so of course we're going to
write some code to do it instead.

Rather than trying to build perl for Fenix, I set up a port forward on the
host VM, and a netcat listener on Fenix, so I could interact with the binary
over the network.

```
root@sphinx:/# sysctl net.ipv4.conf.all.forwarding=1
root@sphinx:/# iptables -t nat -A PREROUTING  -i enp0s3 -p tcp --dport 1234 -j DNAT --to-destination 10.42.0.2
root@sphinx:/# iptables -t nat -A POSTROUTING -o fenix                      -j SNAT --to-source      10.42.0.1

johnny@sphinx # nc -l -p 1234 -e ./sort_sol
```


# The code

I wrote the following perl script to play the game:

```perl
#!/usr/bin/perl

use IO::Socket::INET;

my $S = new IO::Socket::INET(PeerAddr => "10.0.0.90:1234") or die("$!\n");

# 1 | ---------o---------o---------|---------|---------|---------|---------o------
# 2 | ---------|---------o---------o---------o---------o---------o---------|------
# 3 | ---------|---------o---------o---------o---------o---------o---------o------
# 4 | ---------|---------o---------|---------|---------|---------|---------o------
# 5 | ---------o---------|---------|---------|---------|---------o---------o------
# 6 | ---------|---------o---------|---------o---------o---------o---------o------
# 7 | ---------o---------o---------o---------o---------o---------o---------o------

my $flag = "";
my $toggle = "";
my @bits = (0,0,0,0,0,0,0,0);

while (defined($line = $S->getline())) {
	if ($line =~ /([1-7]) ([|o]) ---------([|o])/) {
		$bits[$1] = $3 eq 'o' ? 1 : 0;
		if ($2 ne $3) {
			$toggle .= "$1";
		}
		if ($1 eq '7') {
			$flag .= pack("B*", join("", @bits));
			$S->print("$toggle\r\n");
			print("\n");
			$toggle = "";
		}
	}

	print($line);
}

print "\n$flag\n"
```

The display is occasionally not updated properly, but the flag is still
collected and displayed after the game ends.


## Flag

Flag: `FE{1337_y0uR_f!nG3r5_D0_tHe_w4lKiNg}`


---
_Peter Tirsek, 2020-01-20_
