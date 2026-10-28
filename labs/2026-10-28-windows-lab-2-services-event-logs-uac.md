# Windows Lab #2 – Services, Security event logs, Standard User and PID tracking

**Date:** 28 Oct 2026
**Setup:** Windows 11 VM in UTM, my admin account `ahmetbaykoz`, a temporary Standard User `labstandard`
**Tools:** `services.msc`, `sc`, Event Viewer (`eventvwr.msc`), `net user`, `net session`, `netstat -ano`, `findstr`, `tasklist`

## Goal

In [Windows Lab #1](2026-10-22-windows-lab-1-networking-and-privileges.md) I learned the basic Windows networking and user commands. Yesterday in [Linux Lab #2](2026-10-27-linux-lab-2-processes-services-logs.md) I managed services with `systemctl` and read logs with `journalctl`. This time I wanted the **Windows side of the same ideas**: how services are managed, where authentication events are logged, what a Standard User can and cannot do, and how to trace a network connection back to the program that opened it.

## What I did and what I saw

### 1. Services: `services.msc` and `sc`

I first looked at the services in the **Services** console (`services.msc`). Then I used the command line with the **Windows Update** service. In the GUI it is called "Windows Update", but its real service name is **`wuauserv`**.

```cmd
sc query wuauserv
sc qc wuauserv
```

| Command | What it showed | Meaning |
|---------|----------------|---------|
| `sc query wuauserv` | `STATE: RUNNING` | The service is running **now** |
| `sc qc wuauserv` | `START_TYPE: DEMAND_START` | Its configuration: Windows starts it **when it is needed** (manual), not automatically at boot |

This matches what I saw on Linux: `sc query` is like `systemctl status` (the current state), and the start type in `sc qc` is like `enabled` / `disabled` in systemd (what happens at boot).

### 2. Security event logs: 4624 and 4625

I opened **Event Viewer** (`eventvwr.msc`) → **Windows Logs → Security** and filtered the log by Event IDs **`4624,4625`**.

- **4624** = *An account was successfully logged on*. There were many of these.
- To create a **4625** (*An account failed to log on*) myself, I locked the VM and entered a **wrong password once**. After refreshing Event Viewer, a new 4625 event appeared.

Inside the 4625 event:

| Field | Value | Meaning |
|-------|-------|---------|
| Message | `An account failed to log on` | Failed authentication |
| Logon Type | `2` | **Interactive** logon: someone at the keyboard (not over the network) |
| Account | my own user account | Which account the attempt was for |
| Failure Reason | `Unknown user name or bad password` | |
| Status | `0xC000006D` | Generic "logon failure" |
| Sub Status | `0xC000006A` | The user name was correct, but the **password was wrong** |

Event Viewer plays the same role as `journalctl` on Linux: it records system and security events, and filtering by ID is how you find what matters.

### 3. Standard User vs Administrator

From an **administrator** Command Prompt I created a temporary Standard User:

```cmd
net user labstandard * /add     :: * = prompt for the password instead of typing it on the command line
net user labstandard            :: Local Group Memberships: *Users (not Administrators)
```

Then I signed in as **`labstandard`**:

| Test | Result |
|------|--------|
| Normal CMD → `net session` | ❌ `Access is denied` |
| "Run as administrator" | UAC asked for **administrator credentials** (not just "Yes") |
| After entering my admin account → `whoami` | `ahmetbaykoz` |
| Elevated CMD → `net session` | ✅ worked |

Two things I noticed:

- In Windows Lab #1, as an **admin**, UAC only asked me to **confirm**. As a **Standard User**, UAC asks for an **admin's password** ("over-the-shoulder" elevation).
- After elevation, `whoami` showed **`ahmetbaykoz`**, not `labstandard`. The elevated window runs **as the admin account whose credentials were entered**, not as the Standard User with extra rights. This is similar to `sudo -u` on Linux, where the command really runs as a different user.

### 4. From a network connection to a process

```cmd
netstat -ano
tasklist /FI "PID eq 8508"

netstat -ano | findstr ESTABLISHED
tasklist /FI "PID eq 13256"
```

| Connection | PID | `tasklist` result |
|------------|-----|-------------------|
| A UDP socket from `netstat -ano` | `8508` | `msedgewebview2.exe` |
| An `ESTABLISHED` TCP connection | `13256` | `Widgets.exe` |

So the chain is **connection → PID → process name**. On Linux, `ss -tanp` shows the process directly, or I can match the PID with `ps aux`.

### 5. Cleaning up

```cmd
net user labstandard /delete
net user          :: labstandard is no longer listed
```

The VM is back to how it was before the lab.

## Windows ↔ Linux

| Task | Windows | Linux |
|------|---------|-------|
| Service state now | `sc query <name>` | `systemctl status <name>` |
| Start at boot? | `sc qc <name>` → `START_TYPE` | `systemctl is-enabled <name>` |
| Security / system logs | Event Viewer → Security (4624, 4625) | `journalctl` (and `/var/log/auth.log`) |
| Standard user can't do admin tasks | `Access is denied` → UAC | `Permission denied` → `sudo` |
| Run as another (admin) account | UAC with admin credentials | `sudo` / `sudo -u` |
| Connection → process | `netstat -ano` + `tasklist /FI "PID eq …"` | `ss -tanp` / `ps aux` |
| Create / delete a test user | `net user name * /add` / `/delete` | `useradd -m` / `userdel -r` |

## Security angle

- **Brute-force detection:** many **4625** events in a short time, especially with Sub Status `0xC000006A` (wrong password) for the same account, are a classic sign of password guessing. `0xC0000064` (unknown user name) across many names points to user enumeration. Logon Type tells you *how* it happened: **2** at the keyboard, **3** over the network, **10** Remote Desktop. This is exactly what a SOC analyst or a SIEM rule looks for.
- **Least privilege in practice:** daily work as a **Standard User** means malware launched by mistake cannot make admin changes without a password. The extra credential prompt is a real barrier, not just a click.
- **Services as attack surface and persistence:** attackers often install a malicious service or change the start type of an existing one so their code survives reboots. Checking services with `sc qc` (start type, binary path, account) is a basic incident-response step.
- **Suspicious connections:** `netstat -ano` + `tasklist` is the quickest way to answer "which program is talking to the internet?". A connection owned by an unexpected process is worth investigating.
- **Lab hygiene:** I created the test account with `*` so the password was not saved in the command history, and I deleted it at the end. Leftover test accounts are a common weakness in real systems.

## What I learned

- A service has a **state** (`RUNNING`) and a **configuration** (`START_TYPE`), just like `active` and `enabled` in systemd.
- **4624** = successful logon, **4625** = failed logon. The Logon Type and Sub Status explain *how* and *why*.
- For a Standard User, UAC asks for **admin credentials**, and the elevated process runs **as that admin account**.
- `netstat -ano` → PID → `tasklist` links network activity to a real program.

## Commands used

| Command | Purpose |
|---------|---------|
| `services.msc` | Services console (GUI) |
| `sc query wuauserv` | Current state of a service |
| `sc qc wuauserv` | Service configuration (start type, binary path) |
| `eventvwr.msc` | Event Viewer |
| Filter Current Log → `4624,4625` | Show only successful and failed logons |
| `net user labstandard * /add` | Create a local user (password prompted) |
| `net user labstandard` | Show a user's details and groups |
| `net session` | Admin-only command, used to test elevation |
| `whoami` | Which account the current window runs as |
| `netstat -ano` | Connections and listening ports with PIDs |
| `netstat -ano \| findstr ESTABLISHED` | Only active TCP connections |
| `tasklist /FI "PID eq <PID>"` | Which program owns a PID |
| `net user labstandard /delete` | Delete the test user |

## Next

- Look at **Logon Type 3** and **10** events (network share and Remote Desktop logons) and compare them with type 2.
- Use PowerShell: `Get-Service`, `Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625}` and `Get-NetTCPConnection`.
- Install OpenSSH on the Ubuntu VM and compare SSH failed logins in `journalctl` with Windows 4625 events.
