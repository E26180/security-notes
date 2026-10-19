# Linux Lab #1 – Users, groups and file permissions

**Date:** 19 Oct 2026
**Setup:** Ubuntu VM (UTM), my user `ahmetbaykoz`, a temporary test user `labuser`
**Tools:** `whoami`, `groups`, `id`, `ls -l`, `chmod`, `chown`, `useradd`, `sudo -u`, `userdel`

## Goal

I wanted to understand how Linux decides **who can access a file**: how users and groups are managed, how to read and change permission bits, and what the difference is between `chmod` and `chown`. I tested everything with two real users instead of only reading about it.

## Background I reviewed first

- Every file has an **owner**, a **group** and **permission bits** for three classes: **Owner / Group / Others**.
- `r` = read, `w` = write, `x` = execute, `-` = permission not granted.
- Numeric (octal) form: `r = 4`, `w = 2`, `x = 1`, added up per class. For example `640` = `rw-` / `r--` / `---`.
- **Groups** give several users the same access without setting it user by user.
- Behind the names, Linux tracks users and groups by number: **UID** and **GID**.

## What I did

1. **Checked who I am and which groups I belong to**
   ```bash
   whoami      # ahmetbaykoz
   groups      # ahmetbaykoz ... sudo wireshark docker adm ...
   ```
   Apart from my own group, I am a member of `sudo`, `wireshark`, `docker` and `adm`, among others.

2. **Read permissions on a real file** (in a separate `linux-permissions-lab` folder)
   ```bash
   ls -l test.txt
   # -rw-rw-r--  → owner rw, group rw, others r
   ```

3. **Changed permissions with `chmod`**
   ```bash
   chmod g-w test.txt     # -rw-r--r--  (group loses write)
   chmod 640 test.txt     # -rw-r-----  (owner rw, group r, others nothing)
   ```

4. **Changed ownership with `chown`**
   ```bash
   sudo chown root:root test.txt
   sudo chown ahmetbaykoz:ahmetbaykoz test.txt   # changed back
   ```
   The permission bits stayed the same while the owner and group changed, which made it clear that **permissions and ownership are separate things**.

5. **Created a temporary user**
   ```bash
   sudo useradd -m labuser
   id labuser       # uid=1001(labuser) gid=1001(labuser)
   groups labuser   # labuser
   ```

6. **Tested access from two different users**
   ```bash
   sudo chown labuser:labuser test.txt   # file is still 640
   cat test.txt                          # as ahmetbaykoz → Permission denied
   sudo -u labuser cat test.txt          # as labuser → "Permission test"
   ```

7. **Removed the test user and looked at the file again**
   ```bash
   sudo userdel -r labuser
   ls -l test.txt        # owner and group now shown as 1001 1001
   sudo chown ahmetbaykoz:ahmetbaykoz test.txt
   ```

## What I observed

| Situation | Result | Why |
|-----------|--------|-----|
| `test.txt` is `640`, owned by `labuser:labuser`, read as `ahmetbaykoz` | ❌ Permission denied | I am neither the owner nor in the `labuser` group, and Others = `---` |
| Same file read with `sudo -u labuser` | ✅ Content shown | `labuser` is the owner and has `r` |
| After `userdel -r labuser` | Owner/group shown as `1001 1001` | Ownership is stored as UID/GID; no user record is left to translate `1001` into a name |

**A useful mistake:** I accidentally created a second file, `test.py`. Comparing `ls -l test.py test.txt` showed that `labuser` could read `test.py` because its Others class had `r`, while `test.txt` (`640`) gave Others nothing. The same user got two different answers purely because of the permission bits.

## Security angle

- **Least privilege:** permission bits are the most basic access control in Linux. Secrets and private files should be `600` or `640`, never readable by Others.
- **Group membership is privilege.** Being in `sudo` means I can run anything as root. Membership of `docker` is effectively root-equivalent too, because a container can mount the host filesystem. `wireshark` lets me capture traffic without root, and `adm` lets me read system logs. Auditing who is in these groups is a real security task.
- **Orphaned files:** after deleting a user, their files keep the old UID (`1001`). If a new account is later created with the same UID, it silently becomes the owner of those files. When removing users, their files should be reassigned or cleaned up, as I did with `chown` at the end.

## What I learned

- Access is decided by **owner/group + permission bits together**, not only by "whose file is it?".
- `chmod` changes **what** each class may do; `chown` changes **who** the owner and group are.
- Linux identifies users and groups by **UID/GID** numbers; names are only labels for them.
- Testing with a second real user (`sudo -u`) is the quickest way to prove that a permission setting really works.

## Commands used

| Command | Purpose |
|---------|---------|
| `whoami` | Show the current user |
| `groups` / `groups labuser` | Show group memberships |
| `id labuser` | Show UID, GID and groups of a user |
| `ls -l file` | Show permissions, owner and group |
| `chmod g-w file` | Remove write permission from the group (symbolic) |
| `chmod 640 file` | Set `rw-r-----` (numeric) |
| `sudo chown user:group file` | Change owner and group |
| `sudo useradd -m labuser` | Create a user with a home directory |
| `sudo -u labuser cat file` | Run a command as another user |
| `sudo userdel -r labuser` | Delete a user and their home directory |

## Next

- Directories: what `r`, `w` and `x` mean on a folder (listing vs creating vs entering).
- Special bits: `setuid`, `setgid` and the sticky bit on `/tmp`.
- Check the permissions of `/etc/passwd` and `/etc/shadow` and explain why they differ.
- `umask`: where the default `rw-rw-r--` of a new file comes from.
