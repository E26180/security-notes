# 11 Oct 2026 – Linux and networking commands

Commands I practised while doing [Network Lab #2](../labs/2026-10-11-network-lab-2-ufw-firewall.md), with a short note on what each one does.

## Where am I, who am I?

| Command | What it does |
|---------|--------------|
| `whoami` | Shows the current user |
| `pwd` | Prints the current (working) directory |
| `hostname -I` | Shows the machine's IP address(es) |

## Files and folders

| Command | What it does |
|---------|--------------|
| `ls` | Lists files in a directory |
| `cd <dir>` | Changes directory |
| `mkdir <dir>` | Creates a directory |
| `touch <file>` | Creates an empty file (or updates its timestamp) |
| `cat <file>` | Prints a file's contents |
| `nano <file>` | Opens a simple text editor in the terminal |
| `cp <src> <dst>` | Copies a file |
| `mv <src> <dst>` | Moves or renames a file |
| `rm <file>` | Deletes a file (no recycle bin, so be careful) |

## Networking

| Command | What it does |
|---------|--------------|
| `ip addr` | Shows network interfaces and their IP addresses |
| `ip route` | Shows the routing table, including the default gateway |
| `ping <host>` | Tests whether a host is reachable (ICMP) |
| `ss` | Shows sockets, e.g. which ports are listening (`ss -tuln`) |
| `curl <url>` | Sends an HTTP request from the terminal |

## Searching and logs

| Command | What it does |
|---------|--------------|
| `grep <pattern>` | Finds lines that match a pattern |
| `tail <file>` | Shows the end of a file (`tail -f` follows it live) |
| `cmd1 \| cmd2` | Pipe: sends the output of one command into another |
| `journalctl` | Reads system logs from systemd |

## Admin and firewall

| Command | What it does |
|---------|--------------|
| `sudo <cmd>` | Runs a command with administrator (root) privileges |
| `sudo ufw status` | Shows whether the firewall is active and its rules |
| `sudo ufw default deny incoming` | Blocks all inbound traffic by default |
| `sudo ufw allow 8000/tcp` | Allows inbound TCP traffic on port 8000 |
| `sudo ufw enable` / `disable` | Turns the firewall on or off |

## Takeaway

Most security work on Linux comes down to a few questions: who am I, where am I, what is running, what is listening, and what do the logs say. These commands answer those questions.
