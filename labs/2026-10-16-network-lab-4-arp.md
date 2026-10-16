# Network Lab #4 – ARP: from IP address to MAC address

**Date:** 16 Oct 2026
**Setup:** Ubuntu VM (`enp0s1`, `192.168.64.2/24`, gateway `192.168.64.1`) + macOS host (UTM bridge `bridge100`)
**Tools:** `ip neigh`, `ping`, `tcpdump`, `arp -a`

## Goal

In [Network Lab #3](2026-10-13-network-lab-3-subnetting-dhcp-dns.md) I found that my VM sends off-subnet traffic to the gateway `192.168.64.1`. But an Ethernet frame needs a **MAC address**, not an IP. In this lab I wanted to see exactly how the VM finds the gateway's MAC address, watch it happen on the wire, and understand why this step can be abused.

## Background I reviewed first

- **IP address** – Layer 3 (network), the logical address of a device.
- **MAC address** – Layer 2 (data link), identifies a network interface on the local Ethernet segment.
- **ARP (Address Resolution Protocol)** maps an IPv4 address to a MAC address inside the local network.
- **ARP Request** is **broadcast** to `ff:ff:ff:ff:ff:ff` ("who has 192.168.64.1?"), because the sender doesn't know the target's MAC yet.
- **ARP Reply** is **unicast** back to the sender ("192.168.64.1 is at …"), because the request already contained the sender's IP and MAC.
- Answers are kept in an **ARP / neighbour cache** so the device doesn't have to ask before every packet.

## What I did

1. **Read the neighbour table**
   ```bash
   ip neigh
   ```
   `192.168.64.1 dev enp0s1 lladdr c2:c7:db:11:7b:64 STALE` – `lladdr` is the link-layer (MAC) address of the gateway.

2. **Emptied the cache and triggered ARP with traffic**
   ```bash
   sudo ip neigh flush all
   ping -c 1 192.168.64.1
   ip neigh
   ```
   Flushing alone created nothing. The ping (0% packet loss) is what forced the VM to resolve the gateway's MAC again, and the entry came back in the table.

3. **Captured the ARP exchange**
   ```bash
   sudo tcpdump -i enp0s1 arp
   ```
   While flushing and pinging in a second terminal, I saw:
   ```text
   ARP, Request who-has _gateway tell AEB
   ARP, Reply _gateway is-at c2:c7:db:11:7b:64
   ```

4. **Compared with the Mac host**
   ```bash
   arp -a
   ```
   The Mac listed both the gateway (`192.168.64.1`) and my VM (`192.168.64.2`) on `bridge100`, the virtual bridge UTM uses to connect the host and the VM at Layer 2.

## What I observed

| Step | What happened |
|------|---------------|
| Cache empty | No entry for the gateway |
| VM wants to send a ping | MAC unknown → **ARP Request** (broadcast) |
| Gateway answers | **ARP Reply** (unicast) with `c2:c7:db:11:7b:64` |
| Cache updated | `192.168.64.1 → c2:c7:db:11:7b:64` |
| Then | The ICMP Echo Request can finally be sent |

Other things I noticed:

- **`STALE` and `DELAY`** are normal neighbour states in Linux, not errors. `STALE` means the entry is still cached but hasn't been confirmed recently.
- **`(incomplete)`** on the Mac means it asked but hasn't learned a MAC for that IP yet.
- tcpdump showed **no ICMP** packets even though I pinged, because my capture filter was `arp`. ARP and ICMP are different protocols: ARP resolves the Layer 2 address first, then ICMP carries the ping.

## Security angle: ARP spoofing

ARP has **no authentication**: a device generally accepts a reply without proving it came from the real owner of the IP. An attacker on the same local network can send forged replies so that the victim caches:

```text
192.168.64.1 → attacker's MAC   (instead of the real gateway MAC)
```

The attacker is not claiming their own IP; they are claiming to *be the gateway*. If it works, traffic flows **victim → attacker → gateway**, which is the basis of a **man-in-the-middle (MITM)** attack.

Defences I looked at (conceptually):

| Defence | How it helps |
|---------|--------------|
| **Dynamic ARP Inspection (DAI)** | Switch checks ARP messages and drops invalid ones |
| **DHCP Snooping** | Builds trusted IP–MAC–port bindings that DAI can use |
| **Network segmentation** | Smaller broadcast domains limit who can attack whom |
| **HTTPS / VPN** | Do **not** stop ARP spoofing, but keep the data encrypted even if traffic is intercepted |

## What I learned

- ARP is the missing link between "I know the gateway's IP" (Lab #3) and actually sending a frame on the local network.
- Clearing a cache doesn't trigger anything by itself; real traffic does.
- Choosing the right capture filter matters: a filter can hide traffic that is really there.
- Like DHCP and DNS in Lab #3, ARP works because devices **trust** answers on the local network, and that trust is exactly what attackers exploit.

## Commands used

| Command | Purpose |
|---------|---------|
| `ip neigh` | Show the Linux neighbour (ARP) table |
| `sudo ip neigh flush all` | Clear the neighbour cache |
| `ping -c 1 192.168.64.1` | Send one ICMP packet to trigger ARP |
| `sudo tcpdump -i enp0s1 arp` | Capture only ARP packets on the interface |
| `arp -a` (macOS) | Show the Mac's ARP cache |

## Next

- Capture ARP and ICMP together (`arp or icmp`) to see the full order in one view.
- Capture a DHCP lease renewal (DORA) in Wireshark.
- Read more about how switches implement DAI and DHCP Snooping.
