# Windows Lab #1 – Networking commands, users and UAC

**Date:** 22 Oct 2026
**Setup:** Windows 11 VM in UTM (`192.168.64.4/24`, gateway/DHCP/DNS `192.168.64.1`), Command Prompt (normal and "Run as administrator")
**Tools:** `ipconfig /all`, `ping`, `tracert`, `nslookup`, `netstat -ano`, `whoami`, `net user`, `net localgroup`, `net session`

## Goal

So far I had done all my labs on Linux. This time I wanted to find the **Windows equivalents** of the networking and user-management commands I already use, run them on a real Windows machine, and compare the two systems to check whether the ideas underneath are the same.

## What I did and what I saw

### 1. Network configuration: `ipconfig /all`

```cmd
ipconfig /all
```

| Field | Value |
|-------|-------|
| IPv4 address | `192.168.64.4` |
| Subnet mask | `255.255.255.0` (= `/24`) |
| Default gateway | `192.168.64.1` |
| DHCP enabled | Yes – DHCP server `192.168.64.1` |
| DNS server | `192.168.64.1` |
| Physical Address | the interface's MAC address |

This is the same UTM network as my Ubuntu VM (`192.168.64.2`), so the Windows VM is simply another host on the same `/24`. One `ipconfig /all` shows what I normally collect on Linux with `ip addr`, `ip route` and `nmcli device show`.

### 2. Connectivity: `ping`

```cmd
ping 192.168.64.1
ping google.com
```

- **Gateway:** 4 sent, 4 received, **0% loss**, `<1 ms`. The local link to the gateway works.
- **google.com:** the name was first resolved to `142.250.140.101`, then the ICMP replies came back in about **11–18 ms**. A single test proved that **DNS resolution and internet access** both work.

Note: on Windows `ping` stops after 4 packets by default; on Linux it runs until `Ctrl + C` unless I add `-c 4`.

### 3. The route to Google: `tracert`

```cmd
tracert google.com
```

| Hop | What it was |
|-----|-------------|
| 1 | `192.168.64.1` – the UTM gateway |
| 2 | `10.185.64.1` – a private address on my accommodation's network |
| 3–4 | my ISP's network, including a London core router |
| ~5 | Google |

`tracert` works by sending packets with **TTL = 1, 2, 3 …**. Each router that drops a packet because its TTL reached 0 replies with "Time-to-live exceeded", which reveals that hop. I saved this output to compare later with Linux `traceroute` / `tracepath`.

### 4. DNS: `nslookup`

```cmd
nslookup google.com
```

- The output first shows **which DNS resolver** Windows is using.
- `google.com` returned both **IPv4 addresses (A records)** and **IPv6 addresses (AAAA records)**.
- **`Non-authoritative answer`** means the reply came from a resolver/cache, not directly from Google's authoritative name server.

### 5. Connections and ports: `netstat -ano`

```cmd
netstat -ano
```

- `-a` = all connections and listening ports, `-n` = numeric addresses and ports (no name lookups), `-o` = show the **PID** of the owning process.
- **Local Address** = this VM's IP and source port; **Foreign Address** = the remote IP and port.
- Connections to remote port **443** are HTTPS.
- TCP states I saw in real output:

| State | Meaning |
|-------|---------|
| `LISTENING` | A local service is waiting for incoming connections |
| `ESTABLISHED` | An open, active connection |
| `TIME_WAIT` | The connection is closed; the local side waits briefly so delayed packets are not mistaken for a new connection |
| `CLOSE_WAIT` | The remote side closed; the local application has not closed its end yet |

### 6. Users and groups

```cmd
whoami
net user
net localgroup administrators
```

- `whoami` → `WIN-TSKI4NF2JES\ahmetbaykoz`, in **`COMPUTER-NAME\USERNAME`** format (a *local* account on this machine).
- `net user` listed the local accounts: `Administrator`, `ahmetbaykoz`, `ahmetbaykoz2`, `Guest`, `DefaultAccount`, `WDAGUtilityAccount`.
- `net localgroup administrators` confirmed that **my account is in the Administrators group**.

### 7. UAC: being an admin ≠ running as admin

I used `net session`, which only works with full administrator rights, as a test:

| How CMD was opened | `net session` result |
|--------------------|----------------------|
| Normal Command Prompt | ❌ `Access is denied` |
| **Run as administrator** (accepted the UAC prompt) | ✅ `There are no entries in the list` (it worked; there were simply no active sessions) |

Even though my account is in **Administrators**, programs I start normally run with a **standard-user token**. Full admin rights are only used after an explicit **elevation** through the UAC prompt.

## Windows ↔ Linux

| Task | Windows | Linux (what I used before) |
|------|---------|----------------------------|
| IP, mask, gateway, DNS, MAC | `ipconfig /all` | `ip addr`, `ip route`, `nmcli device show` |
| Reachability | `ping` (4 packets by default) | `ping -c 4` |
| Route to a host | `tracert` | `traceroute` / `tracepath` |
| DNS lookup | `nslookup` | `dig`, `nslookup` |
| Connections, ports, process | `netstat -ano` | `ss -ltnp` (`ss -tanp` for all TCP) |
| Current user | `whoami` | `whoami` |
| List users | `net user` | `/etc/passwd`, `id user` |
| Admin group | `net localgroup administrators` | `groups` → `sudo` group |
| Temporary elevation | UAC → "Run as administrator" | `sudo <command>` |

## Security angle

- **Least privilege by default:** UAC means even an admin account normally runs with a standard token, so malware launched by mistake does not automatically get full control. A **Standard User** account is safer for everyday work; Administrators should be kept to a minimum, just as I should audit who is in `sudo` on Linux.
- **Accounts as attack surface:** Built-in accounts like `Administrator` and `Guest` exist on every Windows machine, and on a properly configured system they are **disabled**. Auditing `net user` and `net localgroup administrators` is a basic hardening and incident-response check.
- **`netstat -ano` for defenders:** a `LISTENING` port or an `ESTABLISHED` connection with an unexpected PID is a classic thing to investigate. The PID can be matched to a program in Task Manager (or with `tasklist`).
- **`tracert` and `nslookup` leak structure:** traceroute exposes internal hops and private ranges (`10.x.x.x`), which is why many networks filter ICMP. DNS is still the first step in reconnaissance.

## What I learned

- Windows and Linux use different commands, but **IP, gateway, DNS, routing, processes, users and privileges work on the same principles**.
- Being in the Administrators group and **running elevated** are different things. UAC plays the same role as `sudo`: separate privilege until it is needed.
- `ipconfig /all` is the first command to run when troubleshooting a Windows network, because it shows IP, mask, gateway, DHCP, DNS and MAC in one place.
- `Non-authoritative answer` in `nslookup` is normal: the answer came from a resolver/cache.

## Commands used

| Command | Purpose |
|---------|---------|
| `ipconfig /all` | Full network configuration of every interface |
| `ping 192.168.64.1` / `ping google.com` | Test the local gateway / DNS + internet |
| `tracert google.com` | Show the routers on the path (increasing TTL) |
| `nslookup google.com` | DNS lookup: resolver, A and AAAA records |
| `netstat -ano` | All connections and listening ports with numeric addresses and PIDs |
| `whoami` | Current user (`COMPUTER\user`) |
| `net user` | List local user accounts |
| `net localgroup administrators` | Members of the local Administrators group |
| `net session` | Admin-only command, used to test UAC elevation |

## Next

- **Network Lab #6:** run `traceroute` / `tracepath` on Ubuntu, watch the increasing TTLs and "Time-to-live exceeded" replies in Wireshark, and compare with this `tracert`.
- Create a **Standard User** and see which of these commands fail without elevation.
- Use `tasklist` (or Task Manager) to match `netstat -ano` PIDs to programs.
- Look at Windows Event Viewer → Security log for logon events (the Windows counterpart of `journalctl`).
