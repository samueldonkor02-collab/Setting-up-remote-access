# Setting-up-remote-access
Hands-on lab practising Windows and Linux access control, including NTFS, SMB, RDP permissions, icacls, and chmod.


# Windows & Linux Access Control Lab (NTFS Permissions, SMB Shares, RDP Access, chmod)

`Windows Server 2019` · `Active Directory` · `PowerShell` · `icacls` · `Kali Linux` · `chmod` · `Remote Desktop` · `PuTTY`

## Overview
This lab was hands-on practice with **access control from both the Windows and Linux side** — the kind of "who can touch this file" work that shows up constantly in real environments. On the Windows side I worked in a domain (`structureality.com`) with a Server 2019 host called `PC10`, using both the GUI and PowerShell to control Remote Desktop access, NTFS file permissions, and SMB share permissions. On the Linux side I used a Kali box to go back to basics with `chmod`, in both symbolic and octal form, to make sure I actually understand permission bits rather than just pasting commands.

I kept the mistakes in this write-up on purpose — a mistyped rotation setting or a permission that didn't apply the way I expected teaches more than a clean run-through.

## Objective
Practice controlling access to files and systems from multiple angles: enabling and restricting Remote Desktop on a domain machine, setting NTFS permissions with `icacls`, managing SMB share-level permissions with PowerShell, checking a user's real effective access, and reinforcing Linux file permission fundamentals with `chmod`.

## Environment
- **Windows side:** Windows Server 2019 (`PC10`), domain `ad.structureality.com`, accessed via RDP and PuTTY from a Windows client
- **Linux side:** Kali Linux VM, accessed via SSH (PuTTY) and the lab's browser-based console
- **Shared folder used for testing:** `C:\LABFILES`
- **Test user for permission changes:** `dylan` (domain account `structureality\dylan`)
- **Local admin accounts used to connect:** `Rene`, `jaime`

## Tools I Used

| Tool | What It Does | Why I Used It |
|------|--------------|----------------|
| **PuTTY** | SSH/RDP client | Connected to the Kali box over SSH and opened Remote Desktop sessions to `PC10` |
| **Remote Desktop Connection** | Windows remote GUI access | Logged into `PC10` as different domain users to test access |
| **System Properties → Remote tab** | Controls whether RDP is allowed and who can use it | Enabled Remote Desktop and added specific users to the Remote Desktop Users group |
| **File Explorer → Security tab** | GUI view of NTFS permissions | Checked and edited permissions on `C:\LABFILES` |
| **Advanced Security Settings → Effective Access** | Shows what a specific user can actually do | Verified `dylan`'s real permissions after several changes, instead of assuming |
| **icacls** | Command-line NTFS permission tool | Granted, denied, and removed permissions for `dylan` on a specific file |
| **PowerShell SMB cmdlets** (`New-SmbShare`, `Grant-SmbShareAccess`, `Revoke-SmbShareAccess`, `Get-SmbShareAccess`) | Manage share-level (not NTFS) permissions | Created a share, gave `dylan` Change access, then revoked it |
| **chmod (symbolic and octal)** | Linux permission tool | Rebuilt a file's permission bits from scratch to understand what each mode actually sets |

## What I Did

### Remote Access Setup
1. Connected to the Kali VM over SSH using PuTTY, pointed at its assigned IP on port 22.
2. Accepted the host key fingerprint on first connect (expected — first-time connection with no cached key).
3. Separately, connected to `PC10` over RDP. The first login attempt used domain credentials (`structureality\Rene`); a later one used a different account (`jaime`) to test access under a second identity.
4. Saw the standard "your remote session will be disconnected" notice when minimizing/reconnecting — a reminder that closing the RDP window doesn't kill programs running on the remote side.

### Enabling and Restricting Remote Desktop
1. On `PC10`, opened **System Properties → Remote tab**. It started on **"Don't allow remote connections to this computer."**
2. Switched it to **"Allow remote connections"** with Network Level Authentication left on (the safer default).
3. Used **Select Users** to open the Remote Desktop Users group, which already listed `structureality\Domain Users`, `structureality\LocalAdmin`, and `structureality\Rene`, with a note that `structureality\jaime` already had access separately (likely via the local Administrators group).
4. Added `dylan` the same way — through **Add → Select Users or Groups**, typing the name and using **Check Names** to resolve it against `ad.structureality.com` before confirming.

### NTFS Permissions with icacls
1. Checked baseline permissions on `comptia-logo.jpg` inside `C:\LABFILES` with a plain `icacls` call — Everyone had Modify, Administrators and SYSTEM had Full Control, and Users had Read & Execute.
2. Ran `icacls .\comptia-logo.jpg /deny dylan:R` — this explicitly denies Read, which always wins over any Allow, even an inherited one.
3. Reversed course with `/grant dylan:F`, giving Full Control. Denies and grants can stack up messily, so I ran a plain `icacls` after each change to see the current state rather than assuming.
4. Cleaned up with `/remove:g dylan`, which strips *all* of dylan's explicit grant entries, DENY included, in one command — a good way to reset a file back to its inherited baseline instead of trying to undo each change one by one.
5. Confirmed the final state matched the original baseline.

### Checking Real Access with Effective Access
1. On the `C:\LABFILES` folder, opened **Properties → Security → Advanced → Effective Access**.
2. Selected `dylan` and ran the check. Full Control, Delete subfolders and files, Change permissions, and Take ownership all came back denied and were flagged as limited by "File Permissions" — everything else (read, write, traverse) was allowed.
3. This was the useful part: instead of guessing from the permission list, this tool actually resolves group membership and inheritance for a specific account and tells you the real answer.

### SMB Share Permissions with PowerShell
1. Created a new share on `C:\LABFILES` with `New-SmbShare -Name "LABFILES" -Path "C:\LABFILES" -Description "Share for LABFILES"`.
2. Confirmed it existed alongside the built-in `ADMIN$`, `C$`, and `IPC$` shares using `Get-SmbShare`.
3. Checked the default share permissions with `Get-SmbShareAccess -Name "LABFILES"` — Everyone had Read.
4. Granted `dylan` Change access at the share level with `Grant-SmbShareAccess -Name "LABFILES" -AccountName "dylan" -AccessRight Change`, confirming the prompt.
5. This is a good reminder that **share permissions and NTFS permissions are separate layers** — a user's actual access is whichever one is more restrictive, not just what the share says.
6. Reversed it with `Revoke-SmbShareAccess -Name "LABFILES" -AccountName "dylan"`, which dropped the share back down to just Everyone: Read.

### Linux File Permissions with chmod
Worked through both permission notations on the same test file to see how they map to each other:

1. Started at the default `-rw-r--r--` on `testfile.txt`.
2. `chmod u+x` → added execute for the owner only (`-rwxr--r--`).
3. `chmod g+w` → added write for the group (`-rwxrw-r--`).
4. `chmod 777` → full read/write/execute for everyone (`-rwxrwxrwx`), the "no restrictions at all" mode.
5. `chmod 740` → owner full control, group read-only, others nothing (`-rwx------` — group and other bits landed as expected for an owner-only working file).
6. `chmod 654` → owner read/write, group read, others read/write, deliberately non-standard to prove I understood the bit math rather than pattern-matching a common mode.
7. Separately, set a small script (`demofile.sh`) to `chmod 710` (owner full, group execute-only, others nothing) and ran it successfully with `./demofile.sh`, confirming the execute bit was actually doing something rather than just displaying differently in `ls -l`.

## Skills I Picked Up
- **The difference between share permissions and NTFS permissions**, and that Windows applies whichever is more restrictive — a mistake I could easily see myself making in a real job if I only checked one layer.
- **Reading `icacls` output correctly**, including that `(DENY)` entries override `(F)`/Full Control grants regardless of order.
- **Using Effective Access instead of guessing.** Reading a permissions list and reasoning out someone's actual access is slower and less reliable than just asking the tool that resolves it for you.
- **Octal vs symbolic chmod**, and being able to convert between them instead of memorizing "common" numbers like 755 without knowing what they mean.
- **Cleaning up test permissions properly** (`/remove:g`, `Revoke-SmbShareAccess`) instead of leaving a lab environment in a messier state than I found it.

## How This Applies in the Real World
Almost every access-related incident comes down to one of these layers being wrong — a share permission left too open, an NTFS deny that didn't propagate the way someone expected, or a Remote Desktop group with more members than it should have. Knowing how to check *actual* effective access, rather than trusting a permissions list at face value, is directly useful for troubleshooting and for auditing who can touch what.

## Where I'm Coming From
I'm transitioning into cybersecurity from a background in **healthcare**. I'm currently studying for **CompTIA Security+** and building out hands-on labs like this one, since practical experience with access control, permissions, and remote administration is what my CV is currently missing.

## What I Want to Learn Next
- Group Policy–based Remote Desktop restrictions instead of per-machine settings
- Auditing NTFS permission changes with Windows Event Logs
- Linux ACLs (`setfacl`/`getfacl`) beyond basic `chmod`
- Combining share and NTFS permissions intentionally as a layered control, rather than treating them separately

## Limitations & What I'd Do Differently in Production
- This was done in a disposable lab environment — I wouldn't test `chmod 777` or broad Deny rules on a live system.
- I didn't test how these permission changes behave for a user logged in *during* the change (session caching of tokens can delay when a permission change takes effect).
- In production I'd document each permission change with a ticket/change reference rather than just the command run, for audit purposes.

## Note on this write-up
A handful of the screenshots I sent while putting this together were duplicates from earlier in the conversation (the same `icacls` PowerShell session, the same `chmod` sequence on `testfile.txt`, the `LABFILES` Properties/Security tab, the Effective Access results for `dylan`, the Grant/Revoke `SmbShareAccess` output, and the RDP "session will be disconnected" prompt each showed up more than once). I used the clearest version of each and didn't duplicate steps in the write-up above.

Give a small description 
