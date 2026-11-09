---
title: "Lab 13: Mapped Drives and Shared Folder Permissions (Part 1)"
date: 2026-11-09
---

Lab 13: Mapped Drives and Shared Folder Permissions (Part 1)

Goal

Practise one of the most common help desk tickets: setting up a shared folder, mapping it to a drive letter, and fixing it when it breaks.

The analogy

A shared folder is like a filing room opened up to visitors from other buildings.

- The UNC path (\\ComputerName\ShareName) is the room's street address.
- The mapped drive (S:) is a speed-dial button on your desk. It's a shortcut, not the room itself.
- Share permissions are the security guard at the front door. They only check people coming in over the network.
- NTFS permissions are the lock on the room door. They apply to everyone, local or network.

Over the network, you need to get past both, and your real access is the more restrictive of the two. Common practice is to keep share permissions open and do the real control with NTFS groups (for example a Read group and a Modify group).

Setup (Windows 11 Home VM, admin Command Prompt)

Create the folder and a test file:

```
cd C:\
mkdir C:\LabShare
cd C:\LabShare
echo This is a test file > test.txt
dir /b C:\LabShare
```

[Screenshot: folder and test.txt listed]

Share the folder:

```
net share LabShare=C:\LabShare /grant:Everyone,FULL
```

[Screenshot: LabShare was shared successfully]

Map it as S:

```
net use S: \\%COMPUTERNAME%\LabShare /persistent:yes
net use
dir /b S:\
```

[Screenshot: net use showing S: with status OK]

[Screenshot: S: in File Explorer under This PC]

Useful check commands

- net share: shows all shares on this PC
- net use: shows mapped drives and their status (OK, Disconnected, Unavailable)
- dir /b: lists file names only
- type: prints a text file's contents

Unplanned test: what survives a crash

My VM crashed mid-lab. After restarting, the folder, test file and share were all still there, but the S: drive mapping was gone. A crash is like pulling the power cord, so a mapping that hadn't been properly saved to the user profile can be lost. Fix: check with net use, then remap. In real life, this is the "my drives disappeared after my PC froze" ticket.

Ticket: "My drive has a red X"

A red X means Windows can't reach the address right now. The drive letter still exists and the folder is usually fine.

1. Double-click the drive first. Windows often tries to reconnect before the network is ready at logon, and clicking it reconnects.
2. Type the full UNC path into File Explorer. If it works, the share is fine, so remove and remap the drive. If it fails, check the network, VPN, server or permissions.
3. Working from home? Check the VPN first.
4. If drives are pushed by Group Policy, fix it there. A manual remap may be overwritten at next logon.

Ticket: "It won't connect when I type the path"

I mistyped the share name (LabShar instead of LabShare) and got:

```
System error 67 has occurred.
The network name cannot be found.
```

[Screenshot: error 67]

The error number tells you which half of the path is wrong:

- Error 67 means the computer was found, but the share name doesn't exist. Check the spelling.
- Error 53 means the computer itself couldn't be reached. Check the computer name, network, VPN or firewall.

Ticket: "I can open files but can't save"

If share permissions are Everyone: Full Control, the block is in NTFS. The user probably has Read and Execute but not Modify.

- Check Folder, Properties, Security, Advanced, Effective Access to see exactly what the user ends up with.
- Add the user to the correct group (for example the Modify group).
- Gotcha: the user needs to log off and back on (or reconnect the drive) before new group membership applies.

Coming in Part 2

Hands-on permissions testing with a separate test user account.
