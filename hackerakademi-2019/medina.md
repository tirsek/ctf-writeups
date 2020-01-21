# Medina

We're given a challenge of connecting to `medina.hackerakademi.dk:1337`
in order to retrieve the flag. When doing so, the remote responds with
the first few characters of the flag, but the connection is terminated
before the entire flag is delivered:

```
[tirsek@archlinux ~]$ telnet medina.hackerakademi.dk 1337
Trying 13.53.130.242...
Connected to medina.hackerakademi.dk.
Escape character is '^]'.
FE{Connection closed by foreign host.
```

## The RST

Consulting a tcpdump of the network traffic, we see a TCP RST (reset)
segment being sent after the first 3 bytes, shown as `[R]` below:

```
22:23:28.961458 IP 10.0.0.2.37066 > 13.53.130.242.1337: Flags [S], seq 4292810481, win 64240, options [mss 1460,sackOK,TS val 3218263833 ecr 0,nop,wscale 6], length 0
22:23:29.067402 IP 13.53.130.242.1337 > 10.0.0.2.37066: Flags [S.], seq 1345820646, ack 4292810482, win 26847, options [mss 1460,sackOK,TS val 4036686004 ecr 3218263833,nop,wscale 7], length 0
22:23:29.067479 IP 10.0.0.2.37066 > 13.53.130.242.1337: Flags [.], ack 1, win 1004, options [nop,nop,TS val 3218263939 ecr 4036686004], length 0
22:23:29.182531 IP 13.53.130.242.1337 > 10.0.0.2.37066: Flags [P.], seq 1:2, ack 1, win 210, options [nop,nop,TS val 4036686119 ecr 3218263939], length 1
22:23:29.182595 IP 10.0.0.2.37066 > 13.53.130.242.1337: Flags [.], ack 2, win 1004, options [nop,nop,TS val 3218264054 ecr 4036686119], length 0
22:23:29.683071 IP 13.53.130.242.1337 > 10.0.0.2.37066: Flags [P.], seq 2:3, ack 1, win 210, options [nop,nop,TS val 4036686619 ecr 3218264054], length 1
22:23:29.683138 IP 10.0.0.2.37066 > 13.53.130.242.1337: Flags [.], ack 3, win 1004, options [nop,nop,TS val 3218264555 ecr 4036686619], length 0
22:23:30.183700 IP 13.53.130.242.1337 > 10.0.0.2.37066: Flags [P.], seq 3:4, ack 1, win 210, options [nop,nop,TS val 4036687120 ecr 3218264555], length 1
22:23:30.183768 IP 10.0.0.2.37066 > 13.53.130.242.1337: Flags [.], ack 4, win 1004, options [nop,nop,TS val 3218265055 ecr 4036687120], length 0
22:23:30.200419 IP 13.53.130.242.1337 > 10.0.0.2.37066: Flags [R], seq 1345820650, win 210, options [nop,nop,TS val 4036687120 ecr 3218264555], length 0
```

This RST segment causes the TCP/IP stack on the local end to consider
the connection to be closed. The challenge also states that the
communication is disrupted by a hostile actor, so why don't we simply
ignore RST segments on this port? We can do that with Linux' iptables:

```
[root@archlinux ~]# iptables -A INPUT -p tcp --sport 1337 -m tcp --tcp-flags RST RST -j DROP
```


## The FIN

After dealing with the RST segment, we run into a similar problem with
a FIN segment, which is transmitted after sent after half the flag is
delivered:

```
22:23:35.189844 IP 13.53.130.242.1337 > 10.0.0.2.37066: Flags [P.], seq 13:14, ack 1, win 210, options [nop,nop,TS val 4036692126 ecr 3218269561], length 1
22:23:35.189943 IP 10.0.0.2.37066 > 13.53.130.242.1337: Flags [.], ack 14, win 1004, options [nop,nop,TS val 3218270062 ecr 4036692126], length 0
22:23:35.204618 IP 13.53.130.242.1337 > 10.0.0.2.37066: Flags [F.], seq 14, ack 1, win 210, options [nop,nop,TS val 4036692126 ecr 3218269561], length 0
```

Ignoring this with another iptables rule ensures the connection stays
open long enough for us to receive the entire flag with no further
interruptions:

```
[root@archlinux ~]# iptables -A INPUT -p tcp --sport 1337 -m tcp --tcp-flags FIN FIN -j DROP

[tirsek@archlinux ~]$ telnet medina.hackerakademi.dk 1337
Trying 13.53.130.242...
Connected to medina.hackerakademi.dk.
Escape character is '^]'.
FE{V3lk0Mm3N_TiL_M3dIN4}
```


## Flag

`FE{V3lk0Mm3N_TiL_M3dIN4}`


---
_Peter Tirsek, 2020-01-20_
