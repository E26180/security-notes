# Network Lab #6 – Routing, traceroute and TTL

**Date:** 22 Oct 2026
**Setup:** Ubuntu VM (`enp0s1`, `192.168.64.2/24`, gateway `192.168.64.1`) in UTM
**Tools:** `ip route`, `tracepath`, `traceroute -I`, Wireshark (display filter `icmp`)

## Goal

Earlier today I ran `tracert` on Windows in [Windows Lab #1](2026-10-22-windows-lab-1-networking-and-privileges.md). This time I wanted to understand **how a route to a remote host is discovered**: read the routing table, trace the path to Google on Linux, and watch in Wireshark exactly how TTL makes each router reveal itself.

## Background

- A **hop** is each router a packet passes through on its way to the destination. If a packet goes through three routers, it has made at least three hops.
- Every IPv4 packet has a **TTL (Time to Live)** field. Each router **decreases it by 1**. When it reaches **0**, the router drops the packet and sends back an **ICMP Time Exceeded** message to the sender.
- traceroute uses this on purpose: it sends probes with TTL = 1, 2, 3 … so that each router along the path, in order, drops one probe and identifies itself.

## What I did

### 1. Routing table

```bash
ip route
```

| Route | Meaning |
|-------|---------|
| `default via 192.168.64.1 dev enp0s1` | Anything not on a local network goes to the UTM gateway through `enp0s1` |
| `192.168.64.0/24 dev enp0s1 ... src 192.168.64.2` | The local subnet is reached directly; my source IP is `192.168.64.2` |
| `172.17.0.0/16`, `172.18.0.0/16` | Virtual networks created by Docker (not used in this lab) |

### 2. `tracepath`

```bash
tracepath google.com
```

- Hop 1: `192.168.64.1` (UTM gateway) → hop 2: `10.185.64.1` → routers on my accommodation's ISP network.
- After a few hops the output showed **`no reply`**. Some routers and firewalls simply do not answer these probes. This does **not** mean the connection is broken; my internet worked normally the whole time.

### 3. `traceroute` with ICMP

`traceroute` was not installed, so I installed it first:

```bash
sudo apt install traceroute
traceroute -I google.com
```

With `-I`, traceroute sends **ICMP Echo Requests** instead of its default UDP probes. This time the route **completed all the way to Google**: UTM gateway → ISP network → Google.

### 4. Watching TTL in Wireshark

1. Started a capture on `enp0s1` with the display filter `icmp`.
2. Ran `traceroute -I google.com` again and watched the packets live.

## What I observed

### Outgoing probes: TTL goes up by one

Opening **Internet Protocol Version 4 → Time to Live** in the Echo Request packets showed `1`, then `2`, `3`, `4` and so on, exactly as the theory says.

### Replies: two different ICMP messages

| Reply | ICMP Type / Code | Who sends it | Meaning |
|-------|------------------|--------------|---------|
| **Time-to-live exceeded** | **Type 11, Code 0** | An intermediate router | "Time to live exceeded in transit": the probe's TTL reached 0 at this router, so it was dropped. The sender address of this reply **is** that hop |
| **Echo Reply** | **Type 0, Code 0** | Google | The probe's TTL was big enough to reach the destination, so traceroute knows it has arrived and stops |

### `tracepath` vs `traceroute -I` vs `tracert`

| Tool | Probe type | Result in my lab |
|------|------------|------------------|
| `tracepath` (Linux) | UDP | First hops shown, then `no reply` |
| `traceroute -I` (Linux) | ICMP Echo | Completed to Google |
| `tracert` (Windows, [Windows Lab #1](2026-10-22-windows-lab-1-networking-and-privileges.md)) | ICMP Echo | Completed to Google in about 5 hops |

The first hops matched on both systems (`192.168.64.1` → `10.185.64.1`), as expected for two VMs on the same UTM network. Both ICMP-based tools reached Google, while the UDP-based `tracepath` stopped getting replies. My explanation is that some devices on the path answer or allow ICMP but not UDP probes.

## Security angle

- **Reconnaissance:** traceroute reveals the internal structure of a network, including private ranges (`10.x.x.x`) and the ISP's routers. Attackers use it to map a target, which is why many networks **filter or rate-limit ICMP** and why `no reply` / `* * *` is common.
- **Firewall testing:** different probe types (UDP, ICMP and TCP with `traceroute -T`) can get different results through the same firewall. Comparing them shows **what a firewall allows**, which is useful for defenders checking their own rules.
- **Don't block ICMP blindly:** Type 11 (Time Exceeded) and Type 3 (Destination Unreachable) are needed for troubleshooting and for Path MTU Discovery, which `tracepath` also reports. Hardening should filter ICMP selectively rather than drop all of it.
- **Docker networks in the routing table:** `172.17.0.0/16` and `172.18.0.0/16` show that installed software can quietly add new networks and routes to a host. Reading `ip route` is a quick way to see every network a machine can reach.

## What I learned

- traceroute does not ask routers "which hop are you?". It **raises the TTL step by step** and builds the route from **which router sends back ICMP Time Exceeded** for each TTL.
- **Type 11** means "dropped on the way, here is who I am". **Type 0** means "you reached the destination".
- `no reply` is a property of the router, not proof that the network is down.
- `ip route` decides the first step of every packet: send it directly on the local subnet, or hand it to the default gateway.

## Commands used

| Command / filter | Purpose |
|------------------|---------|
| `ip route` | Show the routing table and default gateway |
| `tracepath google.com` | Trace the route with UDP probes (no root needed) |
| `sudo apt install traceroute` | Install traceroute |
| `traceroute -I google.com` | Trace the route with ICMP Echo probes |
| Wireshark `icmp` | Show only ICMP: Echo Request, Echo Reply, Time Exceeded |

## Next

- Run `traceroute -T -p 443 google.com` (TCP probes) and compare it with the UDP and ICMP results.
- Use `ip route get 8.8.8.8` and `ip route get 192.168.64.1` to see which route the kernel chooses for a single destination.
- Look at the **MTU** value that `tracepath` reports and read about Path MTU Discovery.
