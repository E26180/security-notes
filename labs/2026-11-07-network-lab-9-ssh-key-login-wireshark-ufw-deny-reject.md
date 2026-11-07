# Network Lab #9 – SSH key-based login, SSH handshake in Wireshark and UFW deny vs reject on port 22

**Date:** 7 Nov 2026
**Setup:** Ubuntu VM (`enp0s1`, `192.168.64.2/24`) in UTM as the SSH server, my Mac (`192.168.64.1`) as the client
**Tools:** `systemctl`, `ssh-keygen`, `ssh-copy-id`, `ssh -i`, `journalctl -u ssh`, Wireshark, UFW, `nc`

## Goal

[Network Lab #8](2026-10-31-network-lab-8-ssh-openssh-ufw-auth-logs.md) used SSH with a **password** and read the logs. This lab does three follow-ups listed there:

1. Log in with an **SSH key pair** instead of a password and see how the log line changes.
2. Capture the **SSH handshake in Wireshark** and see what stays readable and what is encrypted.
3. Compare UFW `deny` and `reject` on **port 22** with the results from [Network Lab #7](2026-10-30-network-lab-7-tcp-traceroute-and-ufw-deny-reject.md).

## What I did and what I saw

### 1. Preparing the SSH server

I checked `ssh.socket`, `ssh.service` and UFW on the VM. The SSH service and the socket were both `inactive`, because I had turned them off at the end of Lab #8.

```bash
sudo systemctl start ssh.socket
systemctl is-active ssh.socket        # active
```

This repeats **socket activation**: starting only `ssh.socket` is enough, and systemd starts the SSH service when a connection arrives.

### 2. SSH key-based authentication

On the Mac I already had a personal ED25519 key in `~/.ssh`. I did **not** touch it and created a separate key pair only for this lab:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_lab9     # private key + .pub, with a passphrase
ssh-copy-id -i ~/.ssh/id_ed25519_lab9.pub <user>@192.168.64.2
ssh -i ~/.ssh/id_ed25519_lab9 <user>@192.168.64.2
whoami                                              # the VM user, so the shell is on Ubuntu
```

| Step | What happened | Meaning |
|------|---------------|---------|
| `ssh-keygen` | `id_ed25519_lab9` (private) and `id_ed25519_lab9.pub` (public); I set a passphrase | The private key stays on the client; the passphrase encrypts it on disk |
| `ssh-copy-id` | First password attempt failed, the second worked, then `Number of key(s) added: 1` | `ssh-copy-id` itself logs in **with a password once** to append the public key to `~/.ssh/authorized_keys` on the server |
| `ssh -i ... ` | Login worked, asking for the key passphrase instead of the account password | The server checks the key, not the account password |

Then I read the logs on the VM:

```bash
sudo journalctl -u ssh --since "30 minutes ago"
```

| Log entry | Meaning |
|-----------|---------|
| `Accepted password for <user> from 192.168.64.1` | The `ssh-copy-id` step (password login) |
| `Accepted publickey for <user> from 192.168.64.1 ... ED25519 SHA256:...` | The real key login, with the **fingerprint** of the key that was used |

The fingerprint in the log matched the lab key I had created, so I could confirm that the login used **this key** and not the password.

### 3. SSH handshake in Wireshark

On the VM I started a capture on `enp0s1`, then opened a **new** SSH connection from the Mac and stopped the capture. Display filter: `tcp.port == 22`.

| Stage | What I saw | Readable? |
|-------|------------|-----------|
| TCP handshake | `SYN` (client) → `SYN/ACK` (server) → `ACK` | yes |
| Version exchange | `SSH-2.0-OpenSSH_...` banners from both sides | **yes, plaintext** |
| Key exchange | `Key Exchange Init`, key exchange messages, `New Keys` | algorithm lists are visible, keys are negotiated |
| Session | `Encrypted packet` | **no**, commands and output are unreadable |

TCP and SSH have separate handshakes: the **TCP** handshake only creates the connection, and the **SSH** key exchange then creates the encryption keys. Wireshark still shows **source and destination IP, ports, packet sizes and timing**, but not the commands typed inside the session.

### 4. UFW `deny` on port 22

I enabled UFW (default incoming policy `deny`), confirmed the `allow 22/tcp` rule, and then removed it:

```bash
sudo ufw status numbered
sudo ufw delete allow 22/tcp
```

Test from the Mac with a **new** TCP connection:

```bash
nc -vz -G 3 192.168.64.2 22
```

My first attempt was a mistake: I ran it **inside the SSH session**, so it ran on the Ubuntu VM, whose `nc` does not know the macOS-only `-G` option. I left the session with `exit` and ran the command in the Mac's own terminal.

Result: `Operation timed out`. The packets are silently dropped, and the client waits for an answer that never arrives.

### 5. UFW `reject` on port 22

```bash
sudo ufw reject 22/tcp
sudo ufw status numbered
```

The same `nc` test from the Mac gave `Connection refused`, immediately.

| UFW rule | What the client sees | What happens |
|----------|----------------------|--------------|
| `deny` (or default deny) | `Operation timed out` | packet silently dropped |
| `reject` | `Connection refused` | the host actively answers that the port is closed |

This matches Lab #7: `deny` leaves a scanner or a client waiting, `reject` fails fast but confirms that the host is alive. An **already established** SSH session can survive a firewall change, so firewall rules must be tested with a **new** connection.

### 6. Cleanup

At the end I returned the lab to its starting state:

- Removed the `reject` rule for port 22 and set UFW back to `inactive`.
- Stopped `ssh.service` and `ssh.socket` (both `inactive`). I kept the `openssh-server` package installed.
- On the VM I checked the contents and the fingerprint in `authorized_keys`, removed the `network-lab9` public key line and deleted the temporary backup file.
- On the Mac the lab **public** key file was removed. I later also deleted the lab **private** key (`id_ed25519_lab9`) and confirmed with `ls -l ~/.ssh` that the file is gone. The matching entry in `authorized_keys` on the VM was already removed, so the key cleanup is complete.

## What I observed

- Key login produced `Accepted publickey` with a key **fingerprint**, while the password step produced `Accepted password`.
- `ssh-copy-id` needs one password login to copy the public key.
- The SSH version banner is plaintext; key exchange then leads to encrypted packets.
- On port 22, `deny` ended in a **timeout** and `reject` ended in **`Connection refused`**, the same difference as in Lab #7.

## Security angle

- **Keys beat passwords:** an attacker cannot guess a long private key the way they can brute-force a password. The next hardening step, which I have not done, is to disable password authentication in `sshd_config` once a key login works.
- **The private key is the secret:** it must stay on the client, and a passphrase protects it if the file is stolen. A key that is no longer needed, like this lab key, should be deleted on both sides, which I did.
- **`authorized_keys` is an access list:** whoever has a matching private key can log in. Old lines should be removed, and I checked the fingerprint before deleting the line.
- **What an observer sees:** on the network, SSH hides the commands but not the metadata (IPs, ports, sizes, timing) or the OpenSSH version banner.
- **Detection:** `Accepted publickey` and the fingerprint in `journalctl` show **which key** was used, which is useful evidence in an investigation.
- **Deny vs reject:** a dropped port tells a scanner nothing, a rejected port confirms the host exists. This is a design choice, not a bug.
- **Cleanup is part of the lab:** turning services off and removing temporary keys and rules keeps the VM from being left more open than it was.

## What I learned

- How to generate a separate ED25519 key pair with a passphrase and install the public key with `ssh-copy-id`.
- How a key login looks in `journalctl -u ssh`: `Accepted publickey ... SHA256:...`.
- The order of an SSH connection in Wireshark: TCP handshake → plaintext version banner → key exchange → encrypted packets.
- On port 22, UFW `deny` gives a timeout and `reject` gives `Connection refused`.
- SSH keys, firewall rules and services are managed **independently**, so each needs its own cleanup.

## Commands used

| Command | Purpose |
|---------|---------|
| `sudo systemctl start ssh.socket` | Start SSH on demand |
| `systemctl is-active ssh.socket` | Check the socket state |
| `ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_lab9` | Create a lab key pair |
| `ssh-copy-id -i <key>.pub <user>@192.168.64.2` | Copy the public key to the server |
| `ssh -i <private key> <user>@192.168.64.2` | Log in with a specific key |
| `sudo journalctl -u ssh --since "30 minutes ago"` | Read SSH logs |
| `tcp.port == 22` | Wireshark display filter |
| `sudo ufw delete allow 22/tcp` | Remove the allow rule |
| `sudo ufw reject 22/tcp` | Add a reject rule |
| `sudo ufw status numbered` | List rules with numbers |
| `nc -vz -G 3 192.168.64.2 22` | Test a new TCP connection from the Mac (macOS `-G` timeout) |

## Next

- Disable password authentication in `sshd_config` (keep a second session open while testing) and see how a login without a key fails in the logs.
- Capture the `reject` case in Wireshark and look at the TCP RST packet.
- Scan my **own** VM to compare how a dropped port and a rejected port look to a scanner.
