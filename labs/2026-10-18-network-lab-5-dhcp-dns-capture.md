# Network Lab #5 – DHCP and DNS on the wire

**Date:** 18 Oct 2026
**Setup:** Ubuntu VM (`enp0s1`, `192.168.64.2/24`, gateway/DHCP/DNS `192.168.64.1`) in UTM
**Tools:** Wireshark (display filters `dhcp` and `dns`), `nmcli`, `dig`, `command -v`

## Goal

In [Network Lab #3](2026-10-13-network-lab-3-subnetting-dhcp-dns.md) I found out *where* my VM's network settings come from (DHCP) and *who* answers its DNS queries (`192.168.64.1` behind the local stub `127.0.0.53`). This time I did not want to learn a new topic. I wanted to **see those two protocols as real packets** and check that what I had read matches what actually happens.

## What I did

### 1. DHCP: forcing the VM to talk to the DHCP server

1. Started a Wireshark capture on the VM's real interface `enp0s1` with the display filter `dhcp`.
2. My plan said to renew the lease with `dhclient`, so I first checked whether it exists:
   ```bash
   command -v dhclient
   ```
   It printed nothing, so `dhclient` is **not installed** on this Ubuntu system. The VM's network is managed by NetworkManager, so I used `nmcli` instead.
3. Took the connection down and brought it back up:
   ```bash
   sudo nmcli connection down netplan-enp0s1
   sudo nmcli connection up netplan-enp0s1
   ```
4. As the connection came back, Wireshark showed the DHCP exchange.

### 2. DNS: one query, one answer

1. Changed the Wireshark display filter to `dns`.
2. Ran a lookup in the terminal:
   ```bash
   dig google.com
   ```
3. Watched the DNS Query and DNS Response appear in Wireshark in real time.

## What I observed

### DHCP: Request and ACK, but no Discover or Offer

| Expected (DORA) | Seen in Wireshark? |
|-----------------|--------------------|
| Discover | ❌ |
| Offer | ❌ |
| Request | ✅ |
| ACK | ✅ |

I only captured **DHCP Request** and **DHCP ACK**. The reason is that the VM still remembered its previous lease. Instead of searching for a server from scratch, the client asked straight away to keep the configuration it already knew, and the server confirmed it with an ACK. So in real networks DHCP does **not** always run the full four-step DORA.

Ports, as expected: DHCP server on **UDP 67**, client on **UDP 68**.

### DNS: the stub resolver vs the real traffic

The `dig google.com` output showed:

| Field | Value | Meaning |
|-------|-------|---------|
| `status` | `NOERROR` | The query succeeded |
| `QUERY` | `1` | One question was asked |
| `ANSWER` | `6` | Six answers came back, all `A` records (IPv4 addresses for `google.com`) |
| `SERVER` | `127.0.0.53#53` | The local `systemd-resolved` stub, **not** the real DNS server |

In Wireshark, however, the packets on the wire went from **`192.168.64.2` (my VM) to `192.168.64.1`** over **UDP 53**. Putting the two views together, the path is:

```text
dig → 127.0.0.53 (systemd-resolved stub) → 192.168.64.1 → upstream DNS
```

This confirmed with real packets what I had only seen in `resolvectl status` during Lab #3.

## Security angle

- **DHCP has no authentication.** A client accepts the first valid answer, so a **rogue DHCP server** on the same network could hand out a malicious gateway or DNS server. On managed switches, **DHCP Snooping** only allows DHCP replies from trusted ports.
- **Classic DNS is plaintext UDP.** I could read the exact domain I looked up in Wireshark, and so could anyone else on the path. That means DNS can leak browsing activity and can be targeted by **DNS spoofing**. **DNSSEC** protects the integrity of answers, and **DNS over HTTPS/TLS** encrypts the queries.
- Together with ARP in [Lab #4](2026-10-16-network-lab-4-arp.md), this shows the same pattern: the basic local-network protocols work on **trust**, which is exactly what attackers abuse.

## What I learned

- Seeing the packets turned memorised theory (DORA, query/response) into something I can point at and explain.
- DHCP can skip Discover/Offer when the client already has a lease; a missing packet is not always a failure.
- The DNS server a tool shows can be a local stub. The packet capture shows where the query really goes.
- Before running a command from a guide, check that the tool exists (`command -v`) and adapt to the system's own tools (`nmcli` instead of `dhclient`).

## Commands used

| Command / filter | Purpose |
|------------------|---------|
| `command -v dhclient` | Check whether a command is installed (no output = not installed) |
| `sudo nmcli connection down netplan-enp0s1` | Take the network connection down |
| `sudo nmcli connection up netplan-enp0s1` | Bring it back up (triggers DHCP) |
| `dig google.com` | Detailed DNS lookup: status, question, answers, server |
| Wireshark `dhcp` | Show only DHCP packets |
| Wireshark `dns` | Show only DNS packets |

## Next

- Capture a **full** DORA: release the lease first or use a fresh VM so the client has to start with Discover.
- Run `dig google.com` twice and check whether the second query reaches the wire or is answered from the local cache.
- Look inside the DHCP ACK in Wireshark: lease time, subnet mask, router and DNS options.
