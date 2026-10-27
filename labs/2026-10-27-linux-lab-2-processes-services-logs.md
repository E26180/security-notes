# Linux Lab #2 – Processes, services and system logs

**Date:** 27 Oct 2026
**Setup:** Ubuntu VM (UTM)
**Tools:** `ps`, `top`, `grep`, `kill`, `systemctl`, `journalctl`

## Goal

In [Linux Lab #1](2026-10-19-linux-lab-1-users-groups-permissions.md) I looked at *who* can access a file. This time I wanted to see *what is running* on a Linux system and how it is managed: short-lived programs as **processes** (with a PID), long-running background programs as **services** under systemd, and the **logs** that record what those services did.

## Background

- Every running program is a **process** with a unique **PID**. The kernel tracks its owner, state, CPU and memory use.
- A **service** (daemon) is a process that runs in the background for a long time, usually started at boot. On Ubuntu, services are managed by **systemd** with `systemctl`.
- systemd sends service and system messages to the **journal**, which I can read and filter with `journalctl`.

## What I did

### 1. Listing processes

```bash
ps aux
top
```

`ps aux` gives a snapshot of every process. The columns I focused on:

| Column | Meaning |
|--------|---------|
| `USER` | Which user owns the process |
| `PID` | The process ID |
| `%CPU` / `%MEM` | CPU and memory use |
| `STAT` | Process state (for example `R` running, `S` sleeping) |
| `COMMAND` | The program and its arguments |

`top` shows the same information **live**, plus a summary of how many processes are running or sleeping and overall CPU and RAM use. (`q` to quit.)

### 2. Starting, finding and stopping my own process

```bash
sleep 300 &                     # start a test process in the background
ps aux | grep '[s]leep 300'     # find it and check its PID
kill <PID>                      # stop it
ps aux | grep '[s]leep 300'     # no output: the process is gone
```

- The `&` runs the command in the background, and the shell printed its PID straight away.
- The `[s]leep` pattern is a small trick: it still matches `sleep 300`, but **not** the `grep` command itself, so the output only shows the real process.
- After `kill`, the second search returned nothing, which proved the process had stopped.

### 3. Managing a service with systemctl

My plan was to use `ssh`, but `systemctl status ssh` showed that **no `ssh.service` exists** on this VM (the OpenSSH server is not installed). I used a service that really exists instead: **`cron`**, the scheduler that runs jobs at set times.

```bash
systemctl status cron            # active (running), enabled
sudo systemctl stop cron         # inactive (dead)
sudo systemctl start cron        # active (running) again
sudo systemctl enable cron       # start automatically at boot
```

| Step | State shown |
|------|-------------|
| Before | `active (running)`, `enabled` |
| After `stop` | `inactive (dead)` |
| After `start` | `active (running)` with a **new Main PID** |
| After `enable` | `enabled` (starts at boot) |

The Main PID changed after the restart. A service is not one fixed process: systemd starts a **new process** each time, and the service name is how I keep track of it.

**`start`/`stop` vs `enable`/`disable`:** `start` and `stop` change what is running **now**; `enable` and `disable` change what happens **at the next boot**. A service can be enabled but currently stopped, or running but not enabled.

### 4. Reading logs with journalctl

```bash
journalctl -u cron        # only the logs of the cron service
journalctl -p err -b      # only error-level messages from the current boot
```

- `journalctl -u cron` showed the systemd events from my test: `Started`, `Stopping`, `Deactivated successfully`, `Stopped`. Logs from **different boot sessions** were kept and shown separately.
- `journalctl -p err -b` filtered the current boot to errors only. I saw a PCI message from the virtual hardware and a GNOME Keyring / PAM error. The goal here was not to fix them, but to learn how to filter the journal by **service** (`-u`), **priority** (`-p`) and **boot** (`-b`).

## What I learned

- Programs run as **processes** that I can inspect and control by PID with `ps`, `top` and `kill`.
- Long-running system programs are **services**, managed by systemd with `systemctl`. Restarting a service gives it a new PID.
- `journalctl` is the place to look for what a service did and when, and its filters make large logs usable.
- When a guide assumes something exists (`ssh.service`), check first and adapt, as I did with `dhclient` in [Network Lab #5](2026-10-18-network-lab-5-dhcp-dns-capture.md).

## Security angle

- **Spotting suspicious processes:** an unexpected process, an unusual owner (for example a web server user running a shell), or high CPU use from an unknown program can be signs of malware or a cryptominer. `ps aux` and `top` are the first things to check on a host.
- **Persistence:** attackers often keep access by adding a **cron job** or a **systemd service** so their code restarts after a reboot. Knowing how `cron` and `systemctl enable` work is the basis for finding this (`systemctl list-units --type=service`, `crontab -l`, `/etc/cron.*`).
- **Logs are evidence:** in incident response, `journalctl` shows when a service started, stopped or failed. Filtering by service, priority and boot narrows thousands of lines down to what matters. This is also why attackers try to clear logs, and why logs are often sent to a separate server.
- **Least privilege:** stopping and starting services needed `sudo`. A normal user cannot turn security services on or off, which links back to the privilege separation in Linux Lab #1.

## Commands used

| Command | Purpose |
|---------|---------|
| `ps aux` | Snapshot of all processes |
| `top` | Live view of processes, CPU and memory |
| `sleep 300 &` | Start a test process in the background |
| `ps aux \| grep '[s]leep 300'` | Find a process without matching `grep` itself |
| `kill <PID>` | Ask a process to stop |
| `systemctl status <service>` | Show whether a service is running and enabled |
| `sudo systemctl stop / start <service>` | Stop or start a service now |
| `sudo systemctl enable <service>` | Start a service automatically at boot |
| `journalctl -u <service>` | Logs of one service |
| `journalctl -p err -b` | Error-level logs from the current boot |

## Next

- Install the OpenSSH server (`sudo apt install openssh-server`), connect from my Mac with `ssh`, allow port 22 in UFW, and read `journalctl -u ssh` for successful and failed logins.
- List all services with `systemctl list-units --type=service` and check scheduled jobs with `crontab -l` and `/etc/cron.*`.
- Compare `kill` (SIGTERM) with `kill -9` (SIGKILL).
