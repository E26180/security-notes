# Linux Lab #3 – Bash: pipes, redirection and an SSH log report script

**Date:** 2 Nov 2026
**Setup:** Ubuntu VM (`192.168.64.2`) in UTM, SSH server from [Network Lab #8](2026-10-31-network-lab-8-ssh-openssh-ufw-auth-logs.md), my Mac as the SSH client (`192.168.64.1`)
**Tools:** `journalctl`, `grep`, `awk`, `sort`, `uniq -c`, redirection (`>`, `>>`), `cat`, `mv`, `chmod`, a Bash script

## Goal

In [Network Lab #8](2026-10-31-network-lab-8-ssh-openssh-ufw-auth-logs.md) I read SSH logs by eye and saw `Failed password` and `Accepted password` lines. In [Linux Lab #2](2026-10-27-linux-lab-2-processes-services-logs.md) I learned to filter logs with `journalctl`. This lab turns that manual reading into a **repeatable command line pipeline and a small Bash script**, so the machine counts the failed logins for me. It also covers the basic Bash tools I had not practised yet: pipes, redirection and running a script.

## What I did and what I saw

### 1. Finding the failed logins

```bash
sudo journalctl -u ssh                              # all SSH service logs
sudo journalctl -u ssh | grep "Failed password"     # only the failed password lines
```

The first command prints every SSH log line. A **pipe** (`|`) sends the output of one command into the next command as its input, so `grep` keeps only the lines that contain `Failed password`.

### 2. Extracting the source IP and counting

```bash
sudo journalctl -u ssh | grep "Failed password" | awk '{print $11}' | sort | uniq -c
```

| Part | What it does |
|------|--------------|
| `grep "Failed password"` | Keeps only the failed login lines |
| `awk '{print $11}'` | Prints the 11th field of each line, which is the **source IP address** in this log format |
| `sort` | Puts identical IPs next to each other (`uniq` only counts neighbouring lines) |
| `uniq -c` | Counts how many times each IP appears |

In my current logs, **`192.168.64.1` (my Mac) appeared 1 time**. That is the one wrong password I typed on purpose in Network Lab #8.

### 3. Redirection

```bash
... | uniq -c > failed_ssh_ips.txt     # write the result to a file (overwrites it)
... | uniq -c >> failed_ssh_ips.txt    # add the result at the end of the file (append)
```

`>` **overwrites** the file and `>>` **appends** to it. I repeated this difference on purpose, because using the wrong one can destroy the old content.

### 4. A Bash script

I put the pipeline into a script called `ssh_failed_report.sh`, made it executable and ran it:

```bash
chmod +x ssh_failed_report.sh
./ssh_failed_report.sh
```

The script analysed the failed SSH logins and wrote the number of failures per IP into `failed_ssh_ips.txt` automatically.

### 5. A small mistake and how I fixed it

When I first created the script, I saved the code into a **`.txt` file by mistake** (`failed_ssh_ips.txt`) instead of the `.sh` file. I checked the content with `cat`, saw that it was the script code, and fixed the name:

```bash
cat failed_ssh_ips.txt
mv failed_ssh_ips.txt ssh_failed_report.sh
```

`mv` renames a file when the destination is a new name in the same folder.

## What I observed

- A pipeline is built step by step: each command does **one small job** and the pipe connects them.
- `awk '{print $11}'` depends on the **position** of the field in the log line. It worked for my log format.
- The script needed `chmod +x` before `./ssh_failed_report.sh` would run. A new file is not executable by default.
- The same command line could be run again and again by the script, with no manual reading of logs.

## Security angle

- **Brute-force detection:** many `Failed password` lines from the **same source IP** in a short time is the classic sign of password guessing. A count per IP turns thousands of log lines into a short list an analyst can act on. In my lab there was only one failure, my own test, but the method is the same on a real server.
- **Fragile parsing:** `$11` works only if the line has the expected layout. A failed login for a user that does not exist uses a different wording (`invalid user`), so the IP can move to another field. I want to test this before trusting the script for real analysis.
- **Scripts need permissions:** `chmod +x` gives execute permission. In real systems an executable file that anyone can edit is a risk, because someone could change what it does. I should only make scripts executable when I understand them.
- **Overwrite risk:** `>` can destroy a file silently. When logs or evidence are involved, `>>` or a new file name is safer.
- **Automation helps defenders:** the same idea is behind tools such as fail2ban, which watch authentication logs and block IPs that fail too often. I have not built that, but I now understand the first step.

## What I learned

- A pipe sends the output of one command to the next command.
- `grep` filters lines, `awk` picks a field, `sort | uniq -c` counts how often each value appears.
- `>` overwrites a file, `>>` appends to it.
- `chmod +x` makes a script executable and `./script.sh` runs it.
- `cat` helps me check what is inside a file before I trust its name, and `mv` can also rename a file.

## Commands used

| Command | Purpose |
|---------|---------|
| `sudo journalctl -u ssh` | SSH service logs |
| `grep "Failed password"` | Keep only failed password lines |
| `awk '{print $11}'` | Print the 11th field (source IP in this format) |
| `sort \| uniq -c` | Count identical lines |
| `> file` / `>> file` | Redirect output (overwrite / append) |
| `chmod +x script.sh` | Make a script executable |
| `./script.sh` | Run a script in the current folder |
| `cat file` | Show the content of a file |
| `mv old new` | Rename or move a file |

## Next

- Test what happens with a failed login for a **non-existing user** and fix the field number if the IP moves.
- Add a threshold to the script (print a warning when an IP has more than a set number of failures).
- Add `Accepted password` counts and the date of each event to the report.
- Rewrite the same log report in Python as a small security utility.
