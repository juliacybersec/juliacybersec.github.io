---
title: "Lab 13: Mapped Drives and Shared Folder Permissions (Part 2)"
date: 2026-11-09
---

Lab 13: Mapped Drives and Shared Folder Permissions (Part 2)

Goal

Work through the "I can open files but can't save" ticket for real, using a test user, a Read group and a Modify group, and see why the user often has to log off and back on before a fix works.

Recap from Part 1

- Share permissions are the security guard at the front door. They only check people coming in over the network.
- NTFS permissions are the lock on the room door. They apply to everyone, local or network.
- Over the network, the more restrictive of the two wins.

In Part 1 the share was set to Everyone: Full Control, so in this lab NTFS does all the real work.

Prediction before starting: a user in the Read group can open files but can't save them, because the guard lets them in but the lock only allows reading.

Note: a "group" with NTFS permissions is not the same thing as Group Policy. Group Policy is the separate system that pushes settings (like drive mappings) out to many PCs.

Step 1: Create a test user and two groups

Windows 11 Home has no Local Users and Groups console, so this is all done from an admin Command Prompt.

```
net user labuser [lab password] /add
net localgroup ShareRead /add
net localgroup ShareModify /add
net localgroup ShareRead labuser /add
net localgroup ShareRead
```

[Screenshot: labuser listed as a member of ShareRead]

Side note: if a password is longer than 14 characters, Windows warns that it won't work with systems older than Windows 2000 and asks whether to continue.

Unplanned ticket: forgotten password

I forgot the test user's password, so I reset it from the admin Command Prompt:

```
net user labuser [new lab password]
```

This is one of the most common help desk tasks.

Step 2: Set the NTFS permissions

Check the starting permissions:

```
icacls C:\LabShare
```

[Screenshot: permissions before]

Folders created straight under C:\ inherit Authenticated Users: Modify by default, which means any logged-in user could save there. That needed tightening.

Grant access to just the right accounts, then switch off inheritance:

```
icacls C:\LabShare /grant Administrators:(OI)(CI)F SYSTEM:(OI)(CI)F ShareRead:(OI)(CI)RX ShareModify:(OI)(CI)M
icacls C:\LabShare /inheritance:r
icacls C:\LabShare
```

[Screenshot: permissions after]

What the codes mean:

- F = Full control
- M = Modify
- RX = Read and execute
- (OI)(CI) = object inherit and container inherit, so the permission flows down to files and subfolders

Grant first, remove inheritance second. Doing it the other way round leaves a moment where the folder has almost no permissions, and you could lock yourself out.

Tip: running icacls on its own (no folder path) shows its help screen, which is a handy cheat sheet for the flags. Most Windows commands also show help with /? (for example net use /?).

Step 3: Test as the user

Open a Command Prompt running as the test user:

```
runas /user:labuser cmd
```

In the new window:

```
whoami
type \\%COMPUTERNAME%\LabShare\test.txt
echo Can I save? >> \\%COMPUTERNAME%\LabShare\test.txt
type C:\LabShare\test.txt
echo Can I save? >> C:\LabShare\test.txt
```

Result: both reads worked and both saves came back Access is denied, over the network and locally. This proves NTFS applies even when someone isn't coming in over the network.

[Screenshot: whoami, read working, save denied]

Step 4: The fix, and the log-off gotcha

In the admin window, add the user to the Modify group:

```
net localgroup ShareModify labuser /add
```

Back in the labuser window that was already open:

```
echo Can I save now? >> C:\LabShare\test.txt
whoami /groups | findstr Share
```

Result: still Access is denied, and only ShareRead is listed. The window is still using the access "badge" issued when it opened, so it doesn't know about the new group.

[Screenshot: still denied, ShareModify missing]

findstr works like grep on Linux. It filters the output to only the lines containing the search word, which helps because the whoami /groups table is very wide.

Give the user a fresh badge (the same as logging off and back on):

```
exit
runas /user:labuser cmd
whoami /groups | findstr Share
echo Can I save now? >> C:\LabShare\test.txt
type C:\LabShare\test.txt
```

Result: both ShareRead and ShareModify listed, the save worked, and the new line appeared in the file.

[Screenshot: both groups listed, saved line in the file]

Lesson: after adding someone to a group, ask them to log off and back on (or reconnect the drive) before testing. Many tickets get reopened because this step is skipped.

Small lesson along the way: a short path like test.txt only works from the folder you're in. A full path like C:\LabShare\test.txt works from anywhere. Also, >> creates a new file if the name doesn't match, so a typo can quietly send your changes to a different file.

Step 5: Check with Effective Access

Right-click C:\LabShare, Properties, Security, Advanced, Effective Access, select labuser, View effective access.

This shows exactly what the user ends up with after all their groups are combined. With Modify, labuser can create, write and delete, but can't change permissions or take ownership (only Full Control allows those).

[Screenshot: Effective Access results]

Help desk checklist: "I can open files but can't save"

1. Check share permissions. If they're open, the block is in NTFS.
2. Use Effective Access to see what the user really has.
3. Add the user to the correct group (not individual permissions).
4. Have the user log off and back on, or reconnect the drive.
5. Test again.

Optional clean-up

```
net localgroup ShareModify labuser /delete
net localgroup ShareRead labuser /delete
net user labuser /delete
```
