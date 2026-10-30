# Network Lab #7 – TCP traceroute and UFW deny vs reject

**Date:** 30 Oct 2026
**Setup:** Ubuntu VM (`enp0s1`, `192.168.64.2/24`) in UTM as the server, my Mac as the client
**Tools:** `traceroute` (UDP, ICMP `-I`, TCP `-T`), Wireshark (display filter `tcp.port == 443 || icmp`), `python3 -m http.server`, `nc`, UFW, `journalctl`

## Goal

In [Network Lab #6](2026-10-22-network-lab-6-routing-traceroute-ttl.md) I saw how traceroute uses TTL with UDP and ICMP probes, and in [Network Lab #2](2026-10-11-network-lab-2-ufw-firewall.md) I saw that a blocked port looks like silence. This time I wanted to answer two follow-up questions:

1. Does the **probe type** (UDP, ICMP or TCP) change what traceroute shows?
2. How does a firewall **`deny`** (drop) differ from a **`reject`** for the client, and what do the logs record?

## Background

- `traceroute` always works the same way: it raises the TTL step by step (1, 2, 3 …) and every router that drops a probe at TTL 0 answers with **ICMP Time Exceeded**. Only the **probe packet** changes: UDP by default, ICMP Echo with `-I`, or a TCP SYN with `-T`.
- A TCP probe goes to a real port (here 443, HTTPS). If the destination answers, it replies like a normal server would (SYN/ACK for an open port).
- UFW can **deny** a connection (drop the packet silently) or **reject** it (drop it and tell the sender).

## What I did

### 1. TCP traceroute

```bash
traceroute -T -p 443 google.com        # permission error as a normal user
sudo traceroute -T -p 443 google.com   # works
```

TCP traceroute needs raw sockets, which are **privileged**, so the first run failed with a permission error and I repeated it with `sudo`. The route went through my accommodation's ISP network and London core routers and reached Google.

### 2. Comparing UDP, ICMP and TCP

```bash
traceroute google.com       # UDP (default)
traceroute -I google.com    # ICMP Echo
```

The overall path was similar for all three probe types. Some hops did not answer certain probe types, so I could see `*` marks, different routers, or a different number of hops depending on the method.

### 3. Watching the TCP probes in Wireshark

I captured on `enp0s1` with `tcp.port == 443 || icmp` while running `sudo traceroute -T -p 443 google.com`. I saw:

| Packet | Meaning |
|--------|---------|
| TCP **SYN** to port 443 with TTL 1, 2, 3 … | The probes. Each one expires one router further along the path |
| **ICMP Time Exceeded** from routers | A router dropped the probe at TTL 0 and revealed itself |
| **SYN/ACK** from Google | The probe was large enough to reach the destination, so traceroute knows it has arrived |

This confirmed at packet level what I had only read about in Lab #6: traceroute never asks routers who they are; it sends probes with different TTL values and reads who answers.

### 4. UFW: deny vs reject

On the VM I started a test web server, and from the Mac I tested the port with `nc`:

```bash
python3 -m http.server 8000                     # on the VM
nc -vz -w 3 192.168.64.2 8000                   # on the Mac
```

With UFW enabled, I tried three situations:

| UFW rule for 8000/tcp | What the Mac saw |
|-----------------------|------------------|
| No rule (before UFW was enabled) | Connection succeeded |
| `sudo ufw deny 8000/tcp` | **No answer, timeout** after 3 seconds |
| `sudo ufw reject 8000/tcp` | **`Connection refused` immediately** |

```bash
sudo ufw status numbered      # shows rules with numbers, so I can delete one with its number
```

So `deny` silently **drops** the packet and the client has to wait for its timeout, while `reject` actively **refuses** the connection and the client finds out straight away.

### 5. UFW logs

```bash
sudo ufw logging on
sudo journalctl -k | grep UFW
```

With the default policy (incoming deny) I tried to connect to port 8000 from the Mac and found `UFW BLOCK` entries. Each entry recorded the **source address**, **destination address**, **protocol (TCP)**, **source port** and **destination port 8000**. There were several blocked SYN entries in a row, because TCP **retries the SYN** when it gets no answer.

## What I observed

- TCP traceroute needs `sudo`; ICMP and UDP traceroute worked in the same session without problems.
- Different probe types can give slightly different results through the same network.
- `deny` = timeout, `reject` = immediate `Connection refused`. The difference is visible from the client without looking at the firewall at all.
- The log shows the same blocked attempt several times, matching the TCP retransmissions I saw in [Lab #2](2026-10-11-network-lab-2-ufw-firewall.md).

## Security angle

- **Drop vs reject:** `deny` (drop) gives an attacker or scanner no information and makes scans slower, because every closed port has to time out. `reject` is friendlier for legitimate users (fast error) but tells a scanner immediately that the host is alive and the port is closed. Which one to choose depends on who is likely to send the traffic.
- **Probe type matters for firewall testing:** a firewall that blocks ICMP or UDP may still allow TCP to port 443, so TCP traceroute often goes further. This is useful for a defender to understand what their own rules actually allow, and it is also how an attacker maps a path.
- **Logs are evidence:** `UFW BLOCK` lines with source IP, destination port and protocol are what an analyst reads to spot port scans (many different `DPT` values from one `SRC`) or repeated attempts on one service.
- **Privilege:** raw-socket tools like TCP traceroute need `sudo`. Normal users cannot craft arbitrary packets, which limits what a compromised low-privilege account can do.

## What I learned

- traceroute's method is the same for all probe types; only the probe packet changes (UDP, ICMP Echo, TCP SYN).
- **Time Exceeded** identifies each hop; **SYN/ACK** (TCP) or **Echo Reply** (ICMP) shows that the destination was reached.
- `ufw deny` = silent drop, client times out. `ufw reject` = immediate refusal, client gets `Connection refused`.
- `sudo ufw status numbered` lists rules with numbers, which makes deleting a specific rule safe.
- `journalctl -k | grep UFW` is a quick way to read firewall blocks with all five important fields.

## Commands used

| Command / filter | Purpose |
|------------------|---------|
| `traceroute google.com` | UDP traceroute (default) |
| `traceroute -I google.com` | ICMP Echo traceroute |
| `sudo traceroute -T -p 443 google.com` | TCP SYN traceroute to port 443 (needs root) |
| Wireshark `tcp.port == 443 \|\| icmp` | Show TCP probes and ICMP Time Exceeded replies |
| `python3 -m http.server 8000` | Test web server on the VM |
| `nc -vz -w 3 <VM-IP> 8000` | Test a TCP port from the Mac with a 3 second timeout |
| `sudo ufw deny 8000/tcp` | Silently drop traffic to port 8000 |
| `sudo ufw reject 8000/tcp` | Refuse traffic to port 8000 and tell the client |
| `sudo ufw status numbered` | List rules with numbers |
| `sudo ufw logging on` | Log blocked traffic |
| `sudo journalctl -k \| grep UFW` | Read `UFW BLOCK` entries |

## Next

- Capture the `reject` case in Wireshark and look at the **TCP RST** packet the VM sends back.
- Scan my **own** VM (only my own machine) to compare how a scanner sees a dropped port versus a rejected one.
- Capture a **full** DHCP DORA from a fresh lease, still open from Lab #5.
- Install the OpenSSH server, open port 22 in UFW and read the SSH login logs with `journalctl -u ssh`.
