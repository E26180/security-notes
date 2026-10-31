# Network Lab #8 – SSH: OpenSSH server, UFW port 22 and authentication logs

**Date:** 31 Oct 2026
**Setup:** Ubuntu VM (`enp0s1`, `192.168.64.2/24`) in UTM as the SSH server, my Mac as the client
**Tools:** `apt`, `systemctl`, `ss`, `ssh`, UFW, `journalctl -u ssh`

## Goal

In [Network Lab #7](2026-10-30-network-lab-7-tcp-traceroute-and-ufw-deny-reject.md) I tested how UFW drops or rejects a port. This lab uses a real service on a real port: I wanted to install an **SSH server**, connect from the Mac, control access with **UFW**, and read the **successful and failed logins** in the logs, like the 4624 and 4625 events on Windows in [Windows Lab #2](2026-10-28-windows-lab-2-services-event-logs-uac.md).

## What I did and what I saw

### 1. Install OpenSSH server

```bash
sudo apt update
sudo apt install openssh-server
systemctl status ssh
systemctl status ssh.socket
ss -ltnp | grep ':22'
```

| Check | What I saw | Meaning |
|-------|------------|---------|
| `systemctl status ssh` | `inactive (dead)` and `TriggeredBy: ssh.socket` | Not an error: Ubuntu can start the service **on demand** |
| `systemctl status ssh.socket` | `active (listening)` | systemd waits on port 22 and starts SSH when a connection arrives (**socket activation**) |
| `ss -ltnp \| grep ':22'` | `0.0.0.0:22` and `[::]:22` | Port 22 is really listening on IPv4 and IPv6 |

### 2. First connection (a small mistake)

My first test was from the Ubuntu terminal to its own address `192.168.64.2`. It worked, which proved the server was running, but it was **not** the real goal, because the traffic never left the VM. The real test is **Mac → Ubuntu**.

### 3. Connect from the Mac

```bash
ssh <user>@192.168.64.2
```

On the first connection SSH showed the server's **ED25519 host key fingerprint** and asked for confirmation. After I typed `yes`, the key was saved in the Mac's `known_hosts` file. If the key for this address ever changes, SSH will warn me, which is a defence against someone pretending to be the server (man in the middle).

Then I ran `whoami`, which printed the VM user. The prompt was in the Mac's terminal, but the shell was on the **Ubuntu VM**.

### 4. SSH vs Telnet

| | SSH | Telnet |
|---|-----|--------|
| Connection | unicast, TCP port 22 | unicast, TCP port 23 |
| Data | **encrypted** | **plaintext**, including passwords |

Neither is broadcast. Broadcast is something like an **ARP Request** on the local network, not an SSH connection.

### 5. SSH and UFW

UFW was `inactive` at the start, so the Mac could connect to port 22 without any rule. Then:

```bash
sudo ufw enable                 # default incoming policy: deny
sudo ufw allow 22/tcp
sudo ufw status numbered        # ALLOW IN rule for 22/tcp
```

A **new** SSH connection from the Mac worked with UFW enabled. The important point:

- **Listening on port 22** means the SSH server is ready to accept connections.
- **UFW allowing 22/tcp** means packets are allowed to reach it.

These are two separate things, and both are needed. A stateful firewall can also keep an **already established** session alive after it is enabled, so a firewall rule must be tested with a **new** connection.

### 6. Authentication logs

I left the SSH session and connected again, and this time I entered the **wrong password once**, then the right one. SSH answered the first attempt with `try again`. On the Ubuntu VM:

```bash
sudo journalctl -u ssh --since "10 minutes ago"
```

| Log entry | Meaning |
|-----------|---------|
| `authentication failure` | The password check failed |
| `Failed password for <user> from 192.168.64.1 ...` | Failed login: **user name**, **source IP**, source port |
| `Accepted password for <user> ...` | Successful login |
| `session opened for user <user>` | A session was created for that user |

From these lines I can read the **user name, source IP, temporary source port, result and when the session was opened**.

## What I observed

- The SSH server was active through **socket activation**, and the service itself showed `inactive (dead)` until a connection came.
- With UFW disabled the port was reachable. With UFW enabled the port needed an explicit `allow 22/tcp` rule.
- One wrong password followed by one correct password produced a `Failed password` line and then an `Accepted password` line for the same connection.

## Security angle

- **Brute-force detection:** many `Failed password` lines from one source IP in a short time is the Linux version of many 4625 events on Windows. Real internet-facing SSH servers receive these constantly.
- **Password logins are risky:** key-based login and disabling password authentication are common hardening steps. I have not changed those settings yet.
- **Host keys:** the `known_hosts` warning is a security feature. Typing `yes` without checking the fingerprint trusts the first connection blindly.
- **Least exposure:** open only the ports a service needs. With a default-deny policy, an extra `allow 22/tcp` makes the SSH server reachable, and I can limit it further to a source address or subnet.
- **Plaintext vs encrypted:** Telnet would have sent the password across the network in clear text. SSH protects the session.
- **Logs are evidence:** the user name, source IP, result and session start time are what an analyst needs to reconstruct an incident.

## What I learned

- Installing `openssh-server` starts listening on TCP 22 through `ssh.socket`; `ss -ltnp` proves it.
- `LISTEN` and a firewall `ALLOW` are different layers. Test new rules with a new connection.
- The first SSH connection saves the host key in `known_hosts`.
- `journalctl -u ssh` shows `Failed password`, `Accepted password` and `session opened`.

## Commands used

| Command | Purpose |
|---------|---------|
| `sudo apt install openssh-server` | Install the SSH server |
| `systemctl status ssh` / `ssh.socket` | Service and socket state |
| `ss -ltnp \| grep ':22'` | Confirm that port 22 is listening |
| `ssh <user>@192.168.64.2` | Connect from the Mac |
| `whoami` | Check which machine and user I am on |
| `sudo ufw enable` | Turn the firewall on |
| `sudo ufw allow 22/tcp` | Allow SSH |
| `sudo ufw status numbered` | List rules with numbers |
| `sudo journalctl -u ssh --since "10 minutes ago"` | Read SSH logs |

## Next

- Try `deny` and `reject` for port 22 and compare with Lab #7 (timeout vs `Connection refused`).
- Capture the SSH handshake in Wireshark and see what stays readable (version banner) and what is encrypted.
- Generate an SSH key pair, log in with the key and look at how the log line changes.
- Capture a **full** DHCP DORA from a fresh lease, still open from Lab #5.
