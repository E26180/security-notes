# Windows Lab #3 – NTFS permissions with `icacls` and PowerShell basics

**Date:** 31 Oct 2026
**Setup:** Windows 11 VM in UTM, my admin account, a temporary Standard User `labstandard`
**Tools:** `icacls`, `net user`, `type`, PowerShell (`Get-Service`, `Get-WinEvent`, `Get-NetTCPConnection`, `Get-Process`)

## Goal

In [Windows Lab #2](2026-10-28-windows-lab-2-services-event-logs-uac.md) I looked at services, logon events and Standard Users with CMD tools. In [Linux Lab #1](2026-10-19-linux-lab-1-users-groups-permissions.md) I learned file permissions with `chmod` and `chown`. This lab has two parts:

1. **NTFS permissions:** how Windows decides who can read or write a file (ACLs, inheritance, explicit DENY).
2. **PowerShell:** the PowerShell versions of the CMD commands I already used for services, security events and connections.

## Part 1 – NTFS permissions

### 1. Setup

From an **administrator** Command Prompt I confirmed my account with `whoami`, then made a safe test area:

I created the folder `C:\NTFS-Lab` with a file `test.txt` in it and looked at the ACL:

```cmd
icacls test.txt
```

The starting ACL showed:

| Who | Permission |
|-----|------------|
| Administrators, SYSTEM | Full control (`F`) |
| Users | Read and execute (`RX`) |
| Authenticated Users | Modify (`M`) |

Every entry had **`(I)`**, which means the permission is **inherited** from the parent folder.

### 2. Inheritance

```cmd
icacls test.txt /inheritance:r
icacls test.txt
```

All permissions on the file were inherited, so removing inheritance left the ACL **empty**. The two similar options behave differently:

| Option | What it does |
|--------|--------------|
| `/inheritance:r` | **Removes** the inherited permissions |
| `/inheritance:d` | **Disables** inheritance but **copies** the inherited permissions as explicit ones |

### 3. A controlled ACL

I built the ACL again by hand: Administrators and SYSTEM got Full control, and the `Users` group got only **Read**:

```cmd
icacls test.txt /grant ...     :: Administrators and SYSTEM = F (Full control), Users = R (Read)
```

I created a temporary Standard User and checked its groups:

```cmd
net user labstandard * /add
net user labstandard          :: Local Group Memberships: *Users only
```

### 4. Testing as the Standard User

| Test (as `labstandard`) | Permission on `Users` | Result |
|--------------------------|-----------------------|--------|
| `type test.txt` | Read (`R`) | ✅ content shown |
| `echo ... >> test.txt` | Read (`R`) | ❌ `Access is denied` |
| `echo ... >> test.txt` | Modify (`M`) | ✅ worked; `type` showed the new line |

So **Read allows reading but not writing**, and **Modify** adds writing. I checked with `type` that the new line was really in the file.

### 5. Explicit DENY

I wanted to see that DENY wins over ALLOW.

- First I denied the generic **`(W)` Write** right to `Users`. This turned out to be a **wider set of rights** than I expected, and it could change the read behaviour as well.
- I removed that rule and denied only the specific rights **`WD` (Write Data)** and **`AD` (Append Data)**:

```cmd
icacls test.txt /deny Users:(WD,AD)     :: after removing the generic (W) deny rule
```

Final result for `labstandard`: the file could still be **read**, but **no new data could be written**, even though Modify was still allowed by another entry. An explicit DENY beats ALLOW **only for the access rights where they overlap**.

### 6. Cleaning up

```cmd
icacls test.txt /reset
net user labstandard /delete
net user                      :: labstandard is no longer listed
```

`/reset` put the file back to normal inherited permissions, and the temporary account was removed.

## Part 2 – PowerShell

### 1. Services

```powershell
Get-Service
Get-Service wuauserv
```

The output has `Status`, `Name` and `DisplayName`. `wuauserv` (Windows Update) was `Running`. This is the PowerShell version of `sc query wuauserv`.

### 2. Security events: `Get-WinEvent`

I filtered the Security log for failed logons (Event ID **4625**) and looked at the newest one:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625}
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 1 | Format-List TimeCreated, Id, Message
```

This event was **different** from the one in Lab #2:

| | Lab #2 event | This event |
|---|--------------|-----------|
| Logon Type | `2` (interactive, at the lock screen) | `11` |
| Process | not noted | `consent.exe` (the UAC prompt) |
| Source address | not noted | localhost `::1` |
| Status codes | `0xC000006D` / Sub Status `0xC000006A` (wrong password) | different values (not noted in detail) |

So not every 4625 is a typed wrong password at the lock screen. This one belonged to a **credential check made through the UAC prompt**. I should read the Logon Type, the process and the Status codes together, not only the Event ID.

### 3. Connections and processes: `Get-NetTCPConnection`

```powershell
Get-NetTCPConnection
Get-Process -Id 8176
```

The columns are `LocalAddress`, `LocalPort`, `RemoteAddress`, `RemotePort`, `State` and `OwningProcess`. I saw the states `Listen`, `Established`, `SynSent` and `TimeWait`. The PID of an established connection belonged to `msedgewebview2`.

## CMD ↔ PowerShell ↔ Linux

| Task | CMD | PowerShell | Linux |
|------|-----|------------|-------|
| Service state | `sc query wuauserv` | `Get-Service wuauserv` | `systemctl status` |
| File permissions | `icacls file` | `Get-Acl file` | `ls -l`, `getfacl` |
| Failed logons | Event Viewer 4625 | `Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625}` | `journalctl`, `auth.log` |
| Connection → process | `netstat -ano` → `tasklist` | `Get-NetTCPConnection` → `Get-Process -Id` | `ss -tanp` |

## Security angle

- **Least privilege:** giving `Users` only Read means a normal account cannot change the file. Over-generous entries such as Modify for `Authenticated Users` are a common mistake that lets any logged-in account change files.
- **Inheritance can hide risk:** a permission set on a parent folder flows down to every file. When I audit access I have to check where a permission comes from (`(I)`), not only what it is.
- **DENY is powerful but narrow:** it blocks only the rights it names. Choosing `W` instead of `WD,AD` changed far more than I wanted, so a DENY rule must be tested.
- **Logs need context:** a 4625 can come from a lock-screen wrong password, a UAC prompt, a network logon or Remote Desktop. The Logon Type and the process tell them apart. A SOC analyst needs this to avoid false alarms and to spot real brute force.
- **PowerShell is useful for defenders and attackers:** it can filter large logs and list connections quickly, which is why it is also heavily logged and monitored.

## What I learned

- `icacls` shows and changes ACLs; `(I)` means inherited; `/inheritance:r` removes and `/inheritance:d` copies.
- Read, Modify and Full control give different results for the same user, and an explicit DENY wins only on the rights it names.
- Use **specific** rights (`WD`, `AD`) when testing DENY and clean up with `/reset`.
- `Get-Service`, `Get-WinEvent` and `Get-NetTCPConnection` + `Get-Process` replace the CMD tools I used before.

## Commands used

| Command | Purpose |
|---------|---------|
| `icacls file` | Show the ACL |
| `icacls file /inheritance:r` / `:d` | Remove inherited permissions / copy them as explicit |
| `icacls file /grant user:R` | Add an allow entry (Read) |
| `icacls file /deny user:(WD,AD)` | Deny writing and appending |
| `icacls file /reset` | Back to inherited defaults |
| `net user labstandard * /add` / `/delete` | Create and delete the test user |
| `Get-Service wuauserv` | Service state |
| `Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625}` | Failed logon events |
| `Format-List TimeCreated, Id, Message` | Show event details |
| `Get-NetTCPConnection` | TCP connections with state and PID |
| `Get-Process -Id <PID>` | Which program owns the PID |

## Next

- Compare Logon Type 3 (network) and 10 (Remote Desktop) events with 2 and 11.
- Try `Get-Acl` and `Set-Acl` instead of `icacls`.
- Test the same permission ideas on a folder, with inheritance flowing to new files.
