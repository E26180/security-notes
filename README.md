# Security Notes

My learning log as a BSc Computer Science with Cyber Security student at the University of York.
I use this repo to write up hands-on labs and summarise what I study, in my own words.

## Projects

| Project | Summary |
|---------|---------|
| [Project 1 – Network & Firewall Lab](projects/network-and-firewall-lab/README.md) | Ubuntu VM + Mac home lab: HTTP and TCP, UFW firewall, subnetting/DHCP/DNS and ARP, with diagrams and packet captures |

## Labs

| Date | Lab | Topics |
|------|-----|--------|
| 16 Oct 2026 | [Network Lab #4 – ARP: from IP address to MAC address](labs/2026-10-16-network-lab-4-arp.md) | ARP request/reply, neighbour cache states, `ip neigh`, `tcpdump arp`, ARP spoofing and defences |
| 13 Oct 2026 | [Network Lab #3 – Subnetting, DHCP and DNS on a real host](labs/2026-10-13-network-lab-3-subnetting-dhcp-dns.md) | CIDR and subnet maths, default gateway, DHCP (DORA, lease options), DNS resolvers (`ip`, `resolvectl`, `nmcli`, `dig`) |
| 11 Oct 2026 | [Network Lab #2 – UFW firewall: blocking and allowing a web server](labs/2026-10-11-network-lab-2-ufw-firewall.md) | Default deny, UFW rules and logs, dropped SYN and TCP retransmission in Wireshark |
| 9 Oct 2026 | [Network Lab #1 – HTTP server and TCP handshake in Wireshark](labs/2026-10-09-network-lab-1-http-wireshark.md) | IP, ports, HTTP, TCP three-way handshake, Wireshark |

## Notes

| Date | Notes | Topics |
|------|-------|--------|
| 14 Oct 2026 | [TryHackMe: DNS in Detail and HTTP in Detail](notes/2026-10-14-tryhackme-dns-http-in-detail.md) | Domain structure, DNS records and lookup chain, `nslookup`; URLs, HTTP methods, status codes, headers, cookies |
| 11 Oct 2026 | [Linux and networking commands](notes/2026-10-11-linux-networking-commands.md) | Files, networking (`ip`, `ss`, `curl`), logs (`grep`, `journalctl`), `sudo`, `ufw` |
| 10 Oct 2026 | [TryHackMe Pre Security](notes/2026-10-10-tryhackme-pre-security.md) | Binary/hex, ASCII/Unicode, IP vs MAC, routers/switches, LAN |

## Current focus

Networking fundamentals → Linux → web security (OWASP Top 10)

*Notes only – no flags or room answers from TryHackMe.*
