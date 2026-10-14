# Network Lab #3 – Subnetting, DHCP and DNS on a real host

**Date:** 13 Oct 2026
**Setup:** Ubuntu VM (`enp0s1`, `192.168.64.2/24`) + macOS host
**Tools:** `ip`, `resolvectl`, `nmcli`, `nslookup`, `dig`

## Goal

Connect three ideas that are usually taught separately: how a device **gets** its IP configuration (DHCP), how it decides **where to send** traffic (subnet + default gateway), and how it **resolves names** (DNS). I first practised the theory, then found every value on my own VM from the terminal. This lab builds on [Network Lab #1](2026-10-09-network-lab-1-http-wireshark.md) and [Network Lab #2](2026-10-11-network-lab-2-ufw-firewall.md).

## Background I reviewed first

- **IPv4 and CIDR:** an IPv4 address is 32 bits; the CIDR prefix (e.g. `/24`) says how many bits belong to the network, and the rest belong to hosts.
- **Usable hosts:** `2^n − 2`, where `n` is the number of host bits (one address is the network address, one is broadcast).
- **Private ranges:** `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`. NAT lets these devices reach the internet through a public address.
- **Default gateway:** the router a host uses for any destination **outside** its own subnet. Traffic to the same subnet does not need it.
- **DHCP (DORA):** Discover → Offer → Request → Acknowledge. Server on UDP 67, client on UDP 68. The address is leased for a fixed time.
- **DNS records:** `A` (IPv4), `AAAA` (IPv6), `NS` (name server). `TTL` is how long an answer may be cached.

## Subnetting practice

| Prefix | Host bits | Usable hosts | Block size |
|--------|-----------|--------------|------------|
| `/24` | 8 | 254 | 256 |
| `/26` | 6 | 62 | 64 |
| `/30` | 2 | 2 | 4 |

Worked example: `192.168.1.130/26`. With a block size of 64, the `/26` subnets start at `.0`, `.64`, `.128` and `.192`, so the address belongs to **`192.168.1.128/26`**:

| | Address |
|-|---------|
| Network | `192.168.1.128` |
| First usable | `192.168.1.129` |
| Last usable | `192.168.1.190` |
| Broadcast | `192.168.1.191` |

## What I did

1. **Found the VM's address and subnet**
   ```bash
   ip addr
   ```
   Main interface `enp0s1`, address `192.168.64.2/24` → network `192.168.64.0/24`, broadcast `192.168.64.255`, 254 usable hosts.

2. **Read the routing table**
   ```bash
   ip route
   ```
   `default via 192.168.64.1` → the default gateway is `192.168.64.1`. Traffic to `8.8.8.8` is outside the subnet, so it is sent to `192.168.64.1`, not straight to `8.8.8.8`.

3. **Found the DNS server**
   ```bash
   which resolvectl
   resolvectl status
   ```
   The upstream DNS server is `192.168.64.1`, the same address as the gateway. It does not have to answer queries itself; it can forward them to other resolvers.

4. **Confirmed the configuration came from DHCP**
   ```bash
   nmcli device show enp0s1
   nmcli -f ipv4.method connection show netplan-enp0s1
   nmcli -f DHCP4 device show enp0s1
   ```
   `ipv4.method: auto` means the address is assigned automatically. The DHCP4 options showed what the server handed out:

   | DHCP option | Value |
   |-------------|-------|
   | DHCP server | `192.168.64.1` |
   | IP address | `192.168.64.2` |
   | Subnet mask | `255.255.255.0` |
   | Router (gateway) | `192.168.64.1` |
   | DNS server | `192.168.64.1` |
   | Lease time | `3600` s (1 hour) |

5. **Resolved a name on Ubuntu**
   ```bash
   nslookup google.com
   ```
   The server shown was `127.0.0.53`. This is not the real upstream server; it is the local stub resolver of `systemd-resolved`, which forwards to `192.168.64.1` (confirmed with `resolvectl status`).

6. **Resolved the same name on macOS**
   ```bash
   nslookup google.com
   dig google.com
   ```
   The Mac used a different DNS server (`10.185.64.1`), because it sits on a different network from the VM. In the `dig` output I read `status: NOERROR`, the `A` and `NS` records, the `TTL` and the query time.

## What I observed

| Question | Where I found the answer | Result on my VM |
|----------|--------------------------|-----------------|
| What is my IP and subnet? | `ip addr` | `192.168.64.2/24` |
| Where does off-subnet traffic go? | `ip route` | gateway `192.168.64.1` |
| Who resolves my DNS queries? | `resolvectl status` | `192.168.64.1` (via local stub `127.0.0.53`) |
| Where did this configuration come from? | `nmcli -f DHCP4 ...` | DHCP server `192.168.64.1`, 1-hour lease |

## Mistakes I made and corrected

- I mixed up the **network**, **last usable** and **broadcast** addresses at first (I called `.254` in a `/24` the network address). Correct: `.0` is the network, `.254` the last usable host and `.255` the broadcast.
- When asked which gateway is used to reach `8.8.8.8`, I first answered `8.8.8.8`. The destination is not the gateway: the routing table's `default via 192.168.64.1` line decides where the packet goes next.

## What I learned

- DHCP hands out much more than an IP address: subnet mask, gateway, DNS server and lease time all arrive together.
- The subnet decides whether a host talks directly or through the gateway, and `ip route` shows that decision.
- The DNS server a tool reports is not always the real one. On Ubuntu, `127.0.0.53` is a local stub; `resolvectl status` shows the upstream server.
- **Security angle:** because clients trust whatever DHCP and DNS tell them, a rogue DHCP server or a spoofed DNS answer can silently redirect traffic. Knowing where these values come from is the first step to checking them.

## Next

- Capture a DHCP lease renewal (DORA) in Wireshark or `tcpdump`.
- Do more subnetting practice with `/27`, `/28` and `/29`.
- Lab #4: ARP and how a host finds the MAC address of its gateway.
