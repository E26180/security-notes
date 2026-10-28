# Linux, networking and Git commands

*First written 11 Oct 2026 · updated 28 Oct 2026*

My personal command reference, built up while doing my [network labs](../projects/network-and-firewall-lab/README.md) and [Linux labs](../labs/2026-10-27-linux-lab-2-processes-services-logs.md), plus the Windows equivalents from my [Windows labs](../labs/2026-10-28-windows-lab-2-services-event-logs-uac.md). Every command here is one I have actually used, with a short note on what it does.

## System and user info

| Command | What it does |
|---------|--------------|
| `whoami` | Shows which user I am logged in as |
| `hostname` | Shows the machine's name |
| `hostname -I` | Shows the machine's IP address(es) |
| `uname -a` | Shows kernel and system information |
| `date` | Shows the current date and time |

## Moving around

| Command | What it does |
|---------|--------------|
| `pwd` | Prints the current (working) directory |
| `ls` | Lists files and folders |
| `ls -l` | Long list with permissions, owner, size and date |
| `ls -a` | Includes hidden files (names starting with `.`) |
| `cd folder` | Goes into a folder |
| `cd ..` | Goes up one level |
| `cd ~` | Goes to my home directory |
| `cd /` | Goes to the root of the filesystem |

## Files and folders

| Command | What it does |
|---------|--------------|
| `mkdir folder` | Creates a folder |
| `touch file.txt` | Creates an empty file (or updates its timestamp) |
| `echo "Hello" > file.txt` | Writes to a file and **overwrites** what was there |
| `echo "Hello" >> file.txt` | **Appends** to the end of a file |
| `cat file.txt` | Prints a file's contents |
| `less file.txt` | Reads a long file page by page (`q` to quit) |
| `head file.txt` | Shows the first lines of a file |
| `tail file.txt` | Shows the last lines of a file |
| `tail -n 20 file.txt` | Shows the last 20 lines |
| `nano file.txt` | Opens the file in the Nano editor |
| `cp file.txt copy.txt` | Copies a file |
| `cp -r folder backup` | Copies a folder and everything inside it |
| `mv old.txt new.txt` | Renames a file |
| `mv file.txt folder/` | Moves a file into a folder |
| `rm file.txt` | Deletes a file (no recycle bin) |
| `rm -i file.txt` | Asks for confirmation before deleting |
| `rmdir folder` | Deletes an **empty** folder |
| `rm -r folder` | Deletes a folder and its contents (be careful) |
| `chmod +x script.sh` | Makes a file executable |

## Terminal basics

| Command / key | What it does |
|---------------|--------------|
| `clear` or `Ctrl + L` | Clears the screen |
| `history` | Shows previously used commands |
| `Ctrl + C` | Stops the running command |
| `exit` | Leaves the current shell |
| `logout` | Leaves a login shell |
| `sudo <command>` | Runs a command with administrator (root) privileges |
| `command -v dhclient` | Checks whether a command is installed (prints its path; no output = not installed) |

## Searching and pipes

| Command | What it does |
|---------|--------------|
| `grep "text" file.txt` | Finds lines containing a pattern |
| `cmd1 \| cmd2` | Pipe: sends the output of one command into another |
| `... \| tail -n 20` | Shows only the last 20 lines of a command's output |

## Users, groups and permissions

| Command | What it does |
|---------|--------------|
| `groups` | Groups the current user belongs to |
| `groups labuser` | Groups of another user |
| `id labuser` | UID, GID and groups of a user |
| `chmod g-w file.txt` | Removes write permission from the group (symbolic form) |
| `chmod 640 file.txt` | Sets `rw-r-----` (numeric form: r=4, w=2, x=1) |
| `sudo chown user:group file.txt` | Changes the owner and group of a file |
| `sudo useradd -m labuser` | Creates a user with a home directory |
| `sudo -u labuser <command>` | Runs a command as another user |
| `sudo userdel -r labuser` | Deletes a user and their home directory |

## Processes

| Command | What it does |
|---------|--------------|
| `ps` | Processes in the current shell |
| `ps aux` | All processes on the system, with details |
| `top` | Live view of processes and CPU/memory use |
| `kill PID` | Stops the process with that ID |
| `command &` | Runs a command in the background and prints its PID |
| `ps aux \| grep '[s]leep 300'` | Finds a process without also matching the `grep` itself |

## Services and logs (systemd)

| Command | What it does |
|---------|--------------|
| `systemctl status cron` | Is the service running (`active`) and does it start at boot (`enabled`)? |
| `sudo systemctl stop cron` / `start cron` | Stops or starts the service now |
| `sudo systemctl enable cron` / `disable cron` | Starts it automatically at boot (or not) |
| `journalctl -u cron` | Logs of one service only |
| `journalctl -p err -b` | Only error-level messages from the current boot |

## Networking

| Command | What it does |
|---------|--------------|
| `ip addr` | Interfaces, IP addresses and CIDR prefix |
| `ip route` | Routing table and default gateway |
| `ip neigh` | Neighbour (ARP) table: IP ↔ MAC mappings |
| `sudo ip neigh flush all` | Clears the neighbour cache |
| `arp -a` | ARP table (also works on macOS) |
| `ping google.com` | Tests whether a host is reachable (ICMP) |
| `ping -c 4 google.com` | Sends exactly 4 pings |
| `ping -c 1 192.168.64.1` | One ping to the gateway; triggers ARP if needed |
| `tracepath google.com` | Traces the route with UDP probes (no root needed); `no reply` = that router did not answer |
| `sudo apt install traceroute` | Installs traceroute |
| `traceroute -I google.com` | Traces the route with ICMP Echo probes (TTL 1, 2, 3 …) |
| `ss -ltnp` | Listening TCP ports and the process using them |
| `ss -ltnp \| grep 8000` | Only the listener on port 8000 |
| `curl http://192.168.64.2:8000` | Sends an HTTP request from the terminal |
| `python3 -m http.server 8000` | Serves the current folder over HTTP on port 8000 |
| `python3 -m http.server 8000 --bind 0.0.0.0` | Same, listening on all interfaces |

## DNS and DHCP

| Command | What it does |
|---------|--------------|
| `which resolvectl` | Shows where a command is installed (checks it exists) |
| `resolvectl status` | DNS servers and resolver settings in use |
| `nslookup google.com` | Resolves a name and shows which DNS server answered |
| `dig google.com` | Detailed DNS answer: records, TTL, query time |
| `nmcli device status` | Network devices managed by NetworkManager |
| `nmcli device show enp0s1` | IP, gateway, DNS and routes for one interface |
| `nmcli connection show netplan-enp0s1` | Details of a connection profile |
| `nmcli -f ipv4.method connection show netplan-enp0s1` | Shows whether IPv4 is automatic (DHCP) or manual |
| `nmcli -f DHCP4 device show enp0s1` | What DHCP handed out: IP, mask, gateway, DNS, lease time |
| `sudo nmcli connection down netplan-enp0s1` | Takes the network connection down |
| `sudo nmcli connection up netplan-enp0s1` | Brings it back up (the VM talks to the DHCP server again) |

## Packet capture

| Command | What it does |
|---------|--------------|
| `sudo tcpdump -i enp0s1 arp` | Captures only ARP packets on `enp0s1` |
| `wireshark` | Opens Wireshark from the terminal |
| Wireshark filter `tcp.port == 8000` | Shows only traffic to/from port 8000 |
| Wireshark filter `dhcp` | Shows only DHCP packets (Discover, Offer, Request, ACK) |
| Wireshark filter `dns` | Shows only DNS queries and responses |
| Wireshark filter `icmp` | Shows only ICMP: Echo Request/Reply and Time Exceeded (Type 11) |

## Firewall (UFW) and logs

| Command | What it does |
|---------|--------------|
| `sudo ufw status` | Is the firewall on, and which rules exist |
| `sudo ufw status verbose` | Same, plus the default policies |
| `sudo ufw enable` / `disable` | Turns the firewall on or off |
| `sudo ufw default deny incoming` | Blocks incoming connections by default |
| `sudo ufw default allow outgoing` | Allows outgoing connections by default |
| `sudo ufw allow 8000/tcp` | Allows incoming TCP on port 8000 |
| `sudo ufw delete allow 8000/tcp` | Removes that allow rule |
| `sudo journalctl -k` | Kernel logs |
| `sudo journalctl -k \| grep UFW \| tail -n 20` | The last 20 UFW firewall log lines |

## Git

| Command | What it does |
|---------|--------------|
| `git --version` | Shows the installed Git version |
| `git init` | Turns the current folder into a repository |
| `git clone <repo-url>` | Downloads a remote repository |
| `git status` | Shows changed, staged and untracked files |
| `git add file.txt` / `git add .` | Stages one file / all changes |
| `git commit -m "message"` | Saves the staged changes as a commit |
| `git commit --amend` | Edits the last commit |
| `git log` / `git log --oneline` | Commit history (full / short) |
| `git diff` | Changes that are not staged yet |
| `git diff --staged` | Changes that are staged |
| `git branch` | Lists branches |
| `git branch new-branch` | Creates a branch |
| `git switch new-branch` | Moves to a branch |
| `git switch -c new-branch` | Creates a branch and moves to it |
| `git branch -d new-branch` | Deletes a branch |
| `git remote -v` | Shows the linked remote repositories |
| `git remote add origin <repo-url>` | Links the local repo to GitHub |
| `git push` / `git push origin main` | Sends local commits to the remote |
| `git push -u origin main` | First push: also sets up tracking |
| `git pull` | Downloads remote changes and merges them |
| `git fetch` | Downloads remote changes without merging |
| `git rm file.txt` | Deletes a file and stages the deletion |
| `git rm --cached file.txt` | Stops tracking a file but keeps it on disk |

**My everyday Git flow:** `git status` → `git add .` → `git commit -m "message"` → `git push`

## Windows equivalents

| Windows (CMD) | What it does | Linux equivalent |
|---------------|--------------|------------------|
| `ipconfig /all` | IP, mask, gateway, DHCP, DNS and MAC of every interface | `ip addr`, `ip route`, `nmcli device show` |
| `ping host` | Reachability test (4 packets by default) | `ping -c 4 host` |
| `tracert host` | Routers on the path, using increasing TTL | `traceroute` / `tracepath` |
| `nslookup host` | DNS lookup and which resolver answered | `dig`, `nslookup` |
| `netstat -ano` | All connections and listening ports, numeric, with PID | `ss -tanp` / `ss -ltnp` |
| `whoami` | Current user as `COMPUTER\user` | `whoami` |
| `net user` | Local user accounts | `/etc/passwd`, `id user` |
| `net localgroup administrators` | Members of the Administrators group | `groups` (look for `sudo`) |
| "Run as administrator" (UAC) | Runs a program with full admin rights | `sudo <command>` |
| `net session` | Admin-only command, handy to check whether CMD is elevated | – |
| `sc query wuauserv` | Current state of a service (e.g. `RUNNING`) | `systemctl status` |
| `sc qc wuauserv` | Service configuration, incl. `START_TYPE` | `systemctl is-enabled` |
| Event Viewer → Security, filter `4624,4625` | Successful / failed logons | `journalctl`, `/var/log/auth.log` |
| `net user name * /add` / `net user name /delete` | Create (password prompted) / delete a local user | `useradd -m` / `userdel -r` |
| `netstat -ano \| findstr ESTABLISHED` | Only active TCP connections | `ss -tn state established` |
| `tasklist /FI "PID eq 1234"` | Which program owns a PID | `ps -p 1234` |

## Takeaway

Most security work on Linux comes down to a few questions: who am I, where am I, what is running, what is listening, who am I talking to, and what do the logs say. These commands answer those questions.
