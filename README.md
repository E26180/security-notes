# Security Notes

My learning log as a BSc Computer Science with Cyber Security student at the University of York.
I use this repo to write up hands-on labs and summarise what I study, in my own words.

## Projects

| Project | Summary |
|---------|---------|
| [Project 1 – Network & Firewall Lab](projects/network-and-firewall-lab/README.md) | Ubuntu VM + Mac home lab: HTTP and TCP, UFW firewall, subnetting/DHCP/DNS, ARP and DHCP/DNS packet captures, routing and traceroute (UDP, ICMP, TCP), UFW deny vs reject, SSH and auth logs, with diagrams |

## Labs

| Date | Lab | Topics |
|------|-----|--------|
| 31 Oct 2026 | [Network Lab #8 – SSH: OpenSSH server, UFW port 22 and authentication logs](labs/2026-10-31-network-lab-8-ssh-openssh-ufw-auth-logs.md) | `openssh-server`, `ssh.socket` (socket activation), `ss -ltnp`, `known_hosts` host key, SSH vs Telnet, UFW `allow 22/tcp`, listening vs firewall allow, `journalctl -u ssh` (`Failed password`, `Accepted password`) |
| 31 Oct 2026 | [Windows Lab #3 – NTFS permissions with `icacls` and PowerShell basics](labs/2026-10-31-windows-lab-3-ntfs-permissions-and-powershell.md) | ACL, inheritance (`/inheritance:r` vs `:d`), Read vs Modify, explicit DENY (`WD`, `AD`), `/reset`; `Get-Service`, `Get-WinEvent` 4625 (Logon Type 11, `consent.exe`), `Get-NetTCPConnection` → `Get-Process` |
| 30 Oct 2026 | [Network Lab #7 – TCP traceroute and UFW deny vs reject](labs/2026-10-30-network-lab-7-tcp-traceroute-and-ufw-deny-reject.md) | `traceroute -T -p 443` (needs root) vs UDP and ICMP probes, TCP SYN probes, Time Exceeded and SYN/ACK in Wireshark, UFW `deny` (timeout) vs `reject` (`Connection refused`), `ufw status numbered`, reading `UFW BLOCK` logs |
| 28 Oct 2026 | [Windows Lab #2 – Services, Security event logs, Standard User and PID tracking](labs/2026-10-28-windows-lab-2-services-event-logs-uac.md) | `sc query` / `sc qc` (state vs start type), Event Viewer 4624/4625 with Logon Type and Sub Status, Standard User and UAC credential prompt, `netstat -ano` → PID → `tasklist`, brute-force detection |
| 27 Oct 2026 | [Linux Lab #2 – Processes, services and system logs](labs/2026-10-27-linux-lab-2-processes-services-logs.md) | `ps aux`, `top`, background jobs and `kill`, systemd services with `systemctl` (start/stop vs enable), new PID after restart, `journalctl` filters by service, priority and boot, persistence and logs as evidence |
| 22 Oct 2026 | [Network Lab #6 – Routing, traceroute and TTL](labs/2026-10-22-network-lab-6-routing-traceroute-ttl.md) | `ip route`, hops, `tracepath` vs `traceroute -I`, TTL 1, 2, 3 … in Wireshark, ICMP Type 11 (Time Exceeded) vs Type 0 (Echo Reply), ICMP filtering |
| 22 Oct 2026 | [Windows Lab #1 – Networking commands, users and UAC](labs/2026-10-22-windows-lab-1-networking-and-privileges.md) | `ipconfig /all`, `ping`, `tracert` (TTL), `nslookup` (A/AAAA, non-authoritative), `netstat -ano` and TCP states, local users and Administrators group, UAC elevation vs `sudo` |
| 19 Oct 2026 | [Linux Lab #1 – Users, groups and file permissions](labs/2026-10-19-linux-lab-1-users-groups-permissions.md) | Owner/Group/Others, symbolic and numeric `chmod`, `chown`, UID/GID, testing access as a second user with `sudo -u`, privileged groups, orphaned files |
| 18 Oct 2026 | [Network Lab #5 – DHCP and DNS on the wire](labs/2026-10-18-network-lab-5-dhcp-dns-capture.md) | DHCP Request/ACK without full DORA, DNS query/response in Wireshark, `127.0.0.53` stub vs real resolver, rogue DHCP and DNS spoofing |
| 16 Oct 2026 | [Network Lab #4 – ARP: from IP address to MAC address](labs/2026-10-16-network-lab-4-arp.md) | ARP request/reply, neighbour cache states, `ip neigh`, `tcpdump arp`, ARP spoofing and defences |
| 13 Oct 2026 | [Network Lab #3 – Subnetting, DHCP and DNS on a real host](labs/2026-10-13-network-lab-3-subnetting-dhcp-dns.md) | CIDR and subnet maths, default gateway, DHCP (DORA, lease options), DNS resolvers (`ip`, `resolvectl`, `nmcli`, `dig`) |
| 11 Oct 2026 | [Network Lab #2 – UFW firewall: blocking and allowing a web server](labs/2026-10-11-network-lab-2-ufw-firewall.md) | Default deny, UFW rules and logs, dropped SYN and TCP retransmission in Wireshark |
| 9 Oct 2026 | [Network Lab #1 – HTTP server and TCP handshake in Wireshark](labs/2026-10-09-network-lab-1-http-wireshark.md) | IP, ports, HTTP, TCP three-way handshake, Wireshark |

## Notes

| Date | Notes | Topics |
|------|-------|--------|
| 14 Oct 2026 | [TryHackMe: DNS in Detail and HTTP in Detail](notes/2026-10-14-tryhackme-dns-http-in-detail.md) | Domain structure, DNS records and lookup chain, `nslookup`; URLs, HTTP methods, status codes, headers, cookies |
| 11 Oct 2026 (updated 28 Oct) | [Linux, networking and Git commands](notes/2026-10-11-linux-networking-commands.md) | Windows CMD equivalents, files, users/groups/permissions, processes, services and logs (`systemctl`, `journalctl`), networking (`ip`, `ss`, `curl`, `traceroute`), DNS/DHCP (`dig`, `nmcli`), ARP, `tcpdump`, Wireshark filters, UFW and logs, Git workflow |
| 10 Oct 2026 | [TryHackMe Pre Security](notes/2026-10-10-tryhackme-pre-security.md) | Binary/hex, ASCII/Unicode, IP vs MAC, routers/switches, LAN |

## Current focus

Networking fundamentals → Linux and Windows (users, permissions, processes, services) → web security (OWASP Top 10)

*Notes only – no flags or room answers from TryHackMe.*
# Security Notes

My learning log as a BSc Computer Science with Cyber Security student at the University of York.
I use this repo to write up hands-on labs and summarise what I study, in my own words.

## Projects

| Project | Summary |
|---------|---------|
| [Project 1 – Network & Firewall Lab](projects/network-and-firewall-lab/README.md) | Ubuntu VM + Mac home lab: HTTP and TCP, UFW firewall, subnetting/DHCP/DNS, ARP and DHCP/DNS packet captures, routing and traceroute (UDP, ICMP, TCP), UFW deny vs reject, with diagrams |

## Labs

| Date | Lab | Topics |
|------|-----|--------|
| 30 Oct 2026 | [Network Lab #7 – TCP traceroute and UFW deny vs reject](labs/2026-10-30-network-lab-7-tcp-traceroute-and-ufw-deny-reject.md) | `traceroute -T -p 443` (needs root) vs UDP and ICMP probes, TCP SYN probes, Time Exceeded and SYN/ACK in Wireshark, UFW `deny` (timeout) vs `reject` (`Connection refused`), `ufw status numbered`, reading `UFW BLOCK` logs |
| 28 Oct 2026 | [Windows Lab #2 – Services, Security event logs, Standard User and PID tracking](labs/2026-10-28-windows-lab-2-services-event-logs-uac.md) | `sc query` / `sc qc` (state vs start type), Event Viewer 4624/4625 with Logon Type and Sub Status, Standard User and UAC credential prompt, `netstat -ano` → PID → `tasklist`, brute-force detection |
| 27 Oct 2026 | [Linux Lab #2 – Processes, services and system logs](labs/2026-10-27-linux-lab-2-processes-services-logs.md) | `ps aux`, `top`, background jobs and `kill`, systemd services with `systemctl` (start/stop vs enable), new PID after restart, `journalctl` filters by service, priority and boot, persistence and logs as evidence |
| 22 Oct 2026 | [Network Lab #6 – Routing, traceroute and TTL](labs/2026-10-22-network-lab-6-routing-traceroute-ttl.md) | `ip route`, hops, `tracepath` vs `traceroute -I`, TTL 1, 2, 3 … in Wireshark, ICMP Type 11 (Time Exceeded) vs Type 0 (Echo Reply), ICMP filtering |
| 22 Oct 2026 | [Windows Lab #1 – Networking commands, users and UAC](labs/2026-10-22-windows-lab-1-networking-and-privileges.md) | `ipconfig /all`, `ping`, `tracert` (TTL), `nslookup` (A/AAAA, non-authoritative), `netstat -ano` and TCP states, local users and Administrators group, UAC elevation vs `sudo` |
| 19 Oct 2026 | [Linux Lab #1 – Users, groups and file permissions](labs/2026-10-19-linux-lab-1-users-groups-permissions.md) | Owner/Group/Others, symbolic and numeric `chmod`, `chown`, UID/GID, testing access as a second user with `sudo -u`, privileged groups, orphaned files |
| 18 Oct 2026 | [Network Lab #5 – DHCP and DNS on the wire](labs/2026-10-18-network-lab-5-dhcp-dns-capture.md) | DHCP Request/ACK without full DORA, DNS query/response in Wireshark, `127.0.0.53` stub vs real resolver, rogue DHCP and DNS spoofing |
| 16 Oct 2026 | [Network Lab #4 – ARP: from IP address to MAC address](labs/2026-10-16-network-lab-4-arp.md) | ARP request/reply, neighbour cache states, `ip neigh`, `tcpdump arp`, ARP spoofing and defences |
| 13 Oct 2026 | [Network Lab #3 – Subnetting, DHCP and DNS on a real host](labs/2026-10-13-network-lab-3-subnetting-dhcp-dns.md) | CIDR and subnet maths, default gateway, DHCP (DORA, lease options), DNS resolvers (`ip`, `resolvectl`, `nmcli`, `dig`) |
| 11 Oct 2026 | [Network Lab #2 – UFW firewall: blocking and allowing a web server](labs/2026-10-11-network-lab-2-ufw-firewall.md) | Default deny, UFW rules and logs, dropped SYN and TCP retransmission in Wireshark |
| 9 Oct 2026 | [Network Lab #1 – HTTP server and TCP handshake in Wireshark](labs/2026-10-09-network-lab-1-http-wireshark.md) | IP, ports, HTTP, TCP three-way handshake, Wireshark |

## Notes

| Date | Notes | Topics |
|------|-------|--------|
| 14 Oct 2026 | [TryHackMe: DNS in Detail and HTTP in Detail](notes/2026-10-14-tryhackme-dns-http-in-detail.md) | Domain structure, DNS records and lookup chain, `nslookup`; URLs, HTTP methods, status codes, headers, cookies |
| 11 Oct 2026 (updated 28 Oct) | [Linux, networking and Git commands](notes/2026-10-11-linux-networking-commands.md) | Windows CMD equivalents, files, users/groups/permissions, processes, services and logs (`systemctl`, `journalctl`), networking (`ip`, `ss`, `curl`, `traceroute`), DNS/DHCP (`dig`, `nmcli`), ARP, `tcpdump`, Wireshark filters, UFW and logs, Git workflow |
| 10 Oct 2026 | [TryHackMe Pre Security](notes/2026-10-10-tryhackme-pre-security.md) | Binary/hex, ASCII/Unicode, IP vs MAC, routers/switches, LAN |

## Current focus

Networking fundamentals → Linux and Windows (users, permissions, processes, services) → web security (OWASP Top 10)

*Notes only – no flags or room answers from TryHackMe.*
