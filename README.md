# Capturing and Analyzing Network Traffic with tcpdump

> **Disclaimer:** This project was completed as part of the Google Cybersecurity Certificate (Coursera). The scenario and environment come from the course. The commands, outputs, and explanations are my own documentation of the work I performed.

## Project description
As a network analyst, I used `tcpdump` on a Linux virtual machine to capture and analyze network traffic. I identified the available network interfaces, inspected live traffic, saved a filtered capture of web traffic (TCP port 80) to a packet capture (`.pcap`) file, and then read the file back in two formats to examine individual packets. Capturing and filtering traffic is a core skill for investigating suspicious activity on a network.

## Commands Used
| Command | Purpose |
|---|---|
| `ifconfig` | Show network interfaces and their configuration |
| `tcpdump -D` | List the interfaces available for packet capture |
| `tcpdump -i` | Choose the interface to capture from |
| `tcpdump -v` | Show detailed (verbose) packet information |
| `tcpdump -c` | Stop after capturing a set number of packets |
| `tcpdump -nn` | Don't resolve IP addresses or ports to names |
| `tcpdump port 80` | Filter to a single port (HTTP) |
| `tcpdump -w` | Write captured packets to a file |
| `tcpdump -r` | Read packets from a saved capture file |
| `tcpdump -X` | Show packet contents in hexadecimal and ASCII |
| `curl` | Generate HTTP traffic to capture |
| `&` | Run a command in the background |

## Identify network interfaces
Before capturing, I needed to know which interface to capture from. First I listed the interfaces with `ifconfig`:

```bash
sudo ifconfig
```
Output:
```
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1460
        inet 172.17.0.2  netmask 255.255.0.0  broadcast 172.17.255.255
        ether 02:42:ac:11:00:02  txqueuelen 0  (Ethernet)
        RX packets 772  bytes 14799929 (14.1 MiB)
        TX packets 605  bytes 50691 (49.5 KiB)

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 58  bytes 8941 (8.7 KiB)
        TX packets 58  bytes 8941 (8.7 KiB)
```

The Ethernet interface is the one with the `eth` prefix, so `eth0` is the interface I used for the rest of the lab. `lo` is the loopback interface, which handles traffic the machine sends to itself.

I then listed the interfaces `tcpdump` can capture from. This works even on systems without `ifconfig`:

```bash
sudo tcpdump -D
```
Output:
```
1.eth0 [Up, Running, Connected]
2.any (Pseudo-device that captures on all interfaces) [Up, Running]
3.lo [Up, Running, Loopback]
4.bluetooth-monitor (Bluetooth Linux Monitor) [Wireless]
5.nflog (Linux netfilter log (NFLOG) interface) [none]
6.nfqueue (Linux netfilter queue (NFQUEUE) interface) [none]
7.dbus-system (D-Bus system bus) [none]
8.dbus-session (D-Bus session bus) [none]
```

## Inspect live network traffic
I captured five packets from `eth0` with detailed output:

```bash
sudo tcpdump -i eth0 -v -c5
```
- `-i eth0`: capture from the `eth0` interface
- `-v`: show detailed packet data
- `-c5`: stop after 5 packets

Output (excerpt):
```
tcpdump: listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
20:42:54.159424 IP (tos 0x0, ttl 64, id 44120, offset 0, flags [DF], proto TCP (6), length 122)
    f5ba238de1a7.5000 > nginx-us-central1-c.c.qwiklabs-terminal-vms-prod-00.internal.47362: Flags [P.], cksum 0x58cf (incorrect -> 0x26a7), seq 3641155095:3641155165, ack 908420067, win 499, options [nop,nop,TS val 4268097872 ecr 947355474], length 70
...
5 packets captured
12 packets received by filter
0 packets dropped by kernel
```

How to read a packet line:

| Field | Meaning |
|---|---|
| `20:42:54.159424` | Timestamp of the packet |
| `IP` | Protocol type |
| `tos`, `ttl`, `id`, `offset`, `flags`, `proto`, `length` | IP header details (verbose mode); here the inner protocol is TCP and the packet length is 122 bytes |
| `f5ba238de1a7.5000 > nginx-...internal.47362` | Source and destination, each with a port. The arrow shows the direction of traffic |
| `Flags [P.]` | TCP flags: `P` is push (the packet carries data) and `.` is acknowledgment |
| `cksum` | Checksum used for error detection |
| `seq`, `ack`, `win` | Sequence number, acknowledgment number, and window size |
| `length 70` | Length of the TCP payload in bytes |

Without `-nn`, `tcpdump` converts IP addresses to names. The same capture also contains a UDP packet to `metadata.google.internal.domain` asking for a reverse lookup (`PTR?`) of an address, which is consistent with `tcpdump` generating its own DNS traffic while resolving names. That is why the next captures use `-nn`.

## Capture network traffic to a file
Next I saved a small sample containing only web traffic. The command runs in the background so I could generate traffic in the same terminal:

```bash
sudo tcpdump -i eth0 -nn -c9 port 80 -w capture.pcap &
```
- `-nn`: don't resolve IP addresses or ports to names. This is a best practice for security work because lookup data may not be valid, and name lookups can alert a threat actor that they are being investigated.
- `-c9`: stop after 9 packets
- `port 80`: capture only port 80, the default HTTP port
- `-w capture.pcap`: save the packets to a file
- `&`: run the command in the background

I generated HTTP traffic with `curl`:

```bash
curl opensource.google.com
```
Output:
```
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="https://opensource.google/">here</A>.
</BODY></HTML>
```
```
9 packets captured
10 packets received by filter
0 packets dropped by kernel
```

Then I verified the capture file exists:

```bash
ls -l capture.pcap
```
Output:
```
-rw-r--r-- 1 tcpdump tcpdump 1401 Oct 21 21:16 capture.pcap
[1]+  Done                    sudo tcpdump -i eth0 -nn -c9 port 80 -w capture.pcap
```

The `Done` line shows the background capture finished after collecting its 9 packets.

## Filter the captured packet data
I read the saved file back with `-r`, first with detailed headers (`-v`):

```bash
sudo tcpdump -nn -r capture.pcap -v
```
The capture shows the full web request, step by step. Addresses and ports are numeric because of `-nn` (`172.17.0.2` is my VM, `142.250.152.102` is the web server):

| Packet | Flags | What is happening |
|---|---|---|
| 1 | `[S]` | My VM sends a SYN to start the TCP connection to port 80 |
| 2 | `[S.]` | The server replies with SYN-ACK |
| 3 | `[.]` | My VM sends an ACK, and the connection is established |
| 4 | `[P.]` | My VM sends `GET / HTTP/1.1` with `Host: opensource.google.com` and `User-Agent: curl/7.88.1` |
| 5 | `[.]` | The server acknowledges the request |
| 6 | `[P.]` | The server responds `HTTP/1.1 301 Moved Permanently` with `Location: https://opensource.google/` |
| 7 | `[.]` | My VM acknowledges the response |
| 8-9 | `[F.]` | Both sides close the connection |

That is the TCP three-way handshake, one HTTP request and response, and the connection teardown in 9 packets.

I then viewed the same file in hexadecimal and ASCII with `-X`:

```bash
sudo tcpdump -nn -r capture.pcap -X
```
Excerpt of the HTTP request packet:
```
0x0040:  2e31 0d0a 486f 7374 3a20 6f70 656e 736f  .1..Host:.openso
0x0050:  7572 6365 2e67 6f6f 676c 652e 636f 6d0d  urce.google.com.
0x0060:  0a55 7365 722d 4167 656e 743a 2063 7572  .User-Agent:.cur
0x0070:  6c2f 372e 3838 2e31 0d0a 4163 6365 7074  l/7.88.1..Accept
0x0080:  3a20 2a2f 2a0d 0a0d 0a                   :.*/*....
```

The left side is the raw packet data in hexadecimal, and the right side is the same bytes as ASCII text. Analysts read hex and ASCII to spot patterns or anomalies during malware analysis and forensics.

## Findings
| Question | Answer |
|---|---|
| Command to capture 3 packets on any interface with verbose output | `sudo tcpdump -c3 -i any -v` |
| What `-i` does | Selects the network interface to monitor |
| What `-v` adds | Verbose information |
| Command to list interfaces available for capture | `sudo tcpdump -D` |

- The capture shows that port 80 traffic is unencrypted. The hostname, the `curl` user agent, and the full response are readable in plain text, which is why sensitive data should only travel over HTTPS.
- The 9-packet capture contains a complete connection lifecycle, from handshake to teardown, so a short filtered capture is enough to reconstruct what a client and server said to each other.

## Full Command Sequence
```bash
sudo ifconfig
sudo tcpdump -D
sudo tcpdump -i eth0 -v -c5
sudo tcpdump -i eth0 -nn -c9 port 80 -w capture.pcap &
curl opensource.google.com
ls -l capture.pcap
sudo tcpdump -nn -r capture.pcap -v
sudo tcpdump -nn -r capture.pcap -X
```

## Summary
I identified `eth0` as the interface to monitor, inspected live traffic, and saved a filtered capture of web traffic to a `.pcap` file. I then read the capture back with `-v` to follow a full HTTP request from handshake to teardown, and with `-X` to view the raw packet contents in hex and ASCII. Capturing to a file and analyzing it later is how analysts preserve evidence and investigate network activity without having to catch it live.

## Key Takeaways
- `tcpdump -D` and `ifconfig` both list interfaces, and `tcpdump -D` works on systems without `ifconfig`.
- Use `-c` to limit a capture, and a filter like `port 80` to record only the traffic you care about.
- Use `-nn` in investigations. Name lookups can return misleading data and generate extra network traffic that could tip off an attacker.
- `-w` saves a capture and `-r` reads it back, so one capture can be analyzed in different ways (`-v` for headers, `-X` for contents).
- TCP flags tell the story of a connection: `S` and `S.` start it, `P.` carries data, and `F.` ends it.
- Unencrypted HTTP exposes its contents to anyone who can capture the traffic.
- Outgoing packets captured on the same machine often show `incorrect` checksums, because the network card commonly fills in the checksum after the capture point.

## Tools
- Linux Bash shell
- `tcpdump`, `ifconfig`, `curl`
