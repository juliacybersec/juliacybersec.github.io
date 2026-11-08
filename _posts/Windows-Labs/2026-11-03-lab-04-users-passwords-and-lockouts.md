---
layout: post
title: "Lab 04: Users, Passwords and Lockouts"
date: 2026-11-03
---

3 November 2026

 Goal

"I forgot my password", "I'm locked out" and "I can't install this" are some of the most common help desk tickets. In this lab I created a test user on my Windows 11 VM and worked through each of those tickets: creating the account, resetting a password, locking the account and unlocking it, and managing group membership.

Environment

- Host: gaming laptop
- Hypervisor: VirtualBox 7
- Guest: Windows 11 Home VM (Win11-Lab)

Before starting

I shut down the VM and took a snapshot called Pre-Lab04, so I could roll back if anything went wrong.

Gotcha: Windows 11 Home has no Local Users and Groups

I planned to use `lusrmgr.msc` (Local Users and Groups), but it wouldn't open. I ran `winver` and found my VM is Windows 11 Home, not Pro. Home editions don't include this console, along with other business tools like Group Policy.

That didn't stop the lab. I managed everything from an **administrator Command Prompt** using `net user`, `net accounts` and `net localgroup`, which work on every edition of Windows.

Step 1: Create a test user

```
net user jsmith <password> /add
net user jsmith /logonpasswordchg:yes
net user
net user jsmith
```

These commands created the account, made the user change their password at first sign-in, listed every account on the PC, and showed the new account's details.

Result: jsmith was created and placed in the Users group by default.

Screenshot: net user jsmith output

Why Users and not Administrators? This is the principle of least privilege: people get only the access they need to do their job. Everyday staff should be standard users. Admin rights belong to the IT staff who need them, and best practice is for IT staff to have two accounts, one everyday account and one separate admin account they only use when needed.

Being careful isn't enough protection on its own. Anything that runs while you're logged in runs with your rights, so if an admin opens a malicious attachment, the malware gets admin rights too. A separate admin account limits that.

Step 2: Ticket: "I've forgotten my password"

I signed in as jsmith, and Windows forced a password change because of the flag I'd set. I then signed back in as admin and reset the password as if the user had called the help desk:

```
net user jsmith *
net user jsmith /logonpasswordchg:yes
```

The `*` makes Windows prompt for the new password and hides it as you type, so no one can read it over your shoulder.

Why force a password change at next logon?
- Users often reuse whatever temporary password they're given and never change it. When I worked on a help desk at a bank, temporary passwords were usually simple, so users could get logged on quickly on a busy Monday morning. Forcing a change means that simple password only exists for a few minutes.
- Until the user changes it, the help desk knows the password, so someone else could log in as that user.
- Accountability: only the user should know their password, so that actions on their account can be traced back to them.

Then vs now: today, help desks **verify the caller's identity before resetting a password**, because pretending to be a locked-out staff member is a common social engineering attack. Many workplaces also use self-service password reset, where users confirm a code on their phone and reset their own password.

Step 3: Ticket: "I'm locked out"

I checked the current lockout policy:

```
net accounts
```

Then I lowered the threshold so the lockout would be easy to trigger:

```
net accounts /lockoutthreshold:3
net accounts /lockoutwindow:5
net accounts /lockoutduration:10
```

Gotcha: System error 87

When I first tried to set the lockout duration to 5 minutes, I got "System error 87 has occurred. The parameter is incorrect." I checked `net accounts` and found the observation window was still at its default of 10 minutes.

Windows has a rule: **the lockout duration can't be shorter than the lockout observation window.** A duration of 10 worked. Setting the window first would also have allowed a shorter duration.

Lesson: read the error, check the current settings, and change one thing at a time.

Triggering the lockout

I signed out and entered the wrong password for jsmith four times. Then I signed back in as admin and ran:

```
net user jsmith
```

Result: Account active showed Locked.

Unlocking

Windows 11 Home has no unlock checkbox, so I waited for the 10-minute lockout duration to pass. The account unlocked by itself, and jsmith could sign in with the correct password. In a workplace, this would be done in Active Directory.

Why lock accounts at all, and why not after one wrong password?

- User side: people make honest mistakes, like typos, Caps Lock, or forgetting a recently changed password. Locking after one mistake would flood the help desk.
- Security side: lockout stops brute-force attacks, where an attacker or a script tries thousands of passwords against one account.
-Too low is a risk too: an attacker could deliberately lock out everyone by entering wrong passwords against every account, which is a denial-of-service attack.
- The gap: password spraying tries one common password across many accounts, so no single account hits the threshold. That's why lockout is one layer alongside strong passwords and MFA.

Step 4: Ticket: "I need admin rights to install Zoom"

I practised granting and removing admin rights:

```
net localgroup Administrators
net localgroup Administrators jsmith /add
net localgroup Administrators
net localgroup Administrators jsmith /delete
```

Screenshot: Administrators group before and after

But in a real ticket, I wouldn't give the user admin rights. Instead I would:
- Check that the software is approved.
- Connect to the user's session with remote support (e.g. Quick Assist), with their permission, and enter my admin credentials at the UAC prompt to install it. This is called over-the-shoulder elevation. The user stays a standard user and my password isn't left on their PC.
- In larger workplaces, push the app out centrally or have the user install it from Company Portal.
- If the meeting is in 10 minutes, get them to join Zoom from a web browser instead.

Lessons learned

- Windows Home doesn't have Local Users and Groups, but the `net` commands do the same job on any edition.
- Least privilege: standard accounts for everyday use, admin only when needed.
- Force a password change after every reset, and verify the caller's identity first.
- Lockout policy is a balance between honest mistakes and stopping password guessing.
- Read error messages carefully: error 87 was a settings rule, not a broken command.

 Skills shown

User account management · Password resets · Account lockout policy · Local groups · Least privilege · Windows command line cmd and Powershell
