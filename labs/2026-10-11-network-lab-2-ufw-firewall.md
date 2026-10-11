# Network Lab #2 – UFW firewall: blocking and allowing a web server

**Date:** 11 Oct 2026
**Setup:** Ubuntu VM (server, `192.168.64.2`) + macOS host (client)
**Tools:** Python `http.server`, UFW, `curl`, Safari, Wireshark

## Goal

See exactly what a host firewall changes on the wire: what a client experiences when a port is blocked, what happens when it is allowed, and how the firewall logs the traffic. This lab builds on [Network Lab #1](2026-10-09-network-lab-1-http-wireshark.md).

## Background I reviewed first

- **TCP vs UDP:** TCP is connection-oriented (handshake, acknowledgements, retransmission); UDP sends datagrams without a connection.
- **IP and ports:** the IP address finds the machine; the port finds the service on it.
- **Private vs public IP**, and how **NAT/PAT** lets many private devices share one public IP by translating addresses and ports.
- **Inbound vs outbound** traffic, **default deny**, and **stateful firewalls**, which track connections so replies to allowed outbound traffic are let back in.
- **NAT is not a firewall:** NAT translates addresses; a firewall decides what is allowed.

## What I did

1. **Started the web server on the VM**
   ```bash
   python3 -m http.server 8000
   ```
   I checked that the Mac could reach `http://192.168.64.2:8000` with `curl` and Safari.

2. **Applied a default-deny policy with UFW**
   ```bash
   sudo ufw default deny incoming
   sudo ufw default allow outgoing
   sudo ufw enable
   sudo ufw status verbose
   ```

3. **Tested with port 8000 blocked.** The request from the Mac no longer got a response.

4. **Allowed the port and tested again**
   ```bash
   sudo ufw allow 8000/tcp
   ```
   The page loaded again in both `curl` and Safari.

5. **Read the UFW log entries** and identified the key fields:

   | Field | Meaning |
   |-------|---------|
   | `SRC` | source IP (the Mac) |
   | `DST` | destination IP (the VM) |
   | `SPT` | source port (random, client side) |
   | `DPT` | destination port (`8000`) |
   | `PROTO` | protocol (`TCP`) |

6. **Captured both cases in Wireshark** with the filter `tcp.port == 8000`.

7. **Cleaned up:** I removed the allow rule, disabled the firewall with `sudo ufw disable`, and confirmed `Status: inactive` with `sudo ufw status`.

## What I observed

| Firewall state | Packets in Wireshark | Client result |
|----------------|----------------------|---------------|
| **ALLOW 8000/tcp** | `SYN` → `SYN-ACK` → `ACK` → `GET /` → `200 OK` | Page loads |
| **DENY (default)** | `SYN` only, followed by `TCP Retransmission` | No response |

When the port was blocked, the server never replied. The client kept resending its `SYN` because it never received a `SYN-ACK`, so the TCP handshake could not complete and no HTTP traffic was ever sent.

## What I learned

- A firewall rule acts **before** the application: when it blocks the port, the web server never even sees the request.
- A **dropped** packet looks different from a closed port. The client gets no answer at all and keeps retransmitting, rather than receiving an immediate rejection.
- **Default deny incoming + allow outgoing** is a simple, secure baseline: open only the ports a service actually needs.
- Firewall logs are useful evidence. `SRC`, `DST`, `SPT`, `DPT` and `PROTO` show who tried to connect, to which service, and how.
- It is good practice to **undo lab changes** and verify the final state.

## Next

- Look at the difference between UFW `deny` (silently drop) and `reject` (send a refusal) in Wireshark.
- Practise reading logs with `grep` and `tail`.
