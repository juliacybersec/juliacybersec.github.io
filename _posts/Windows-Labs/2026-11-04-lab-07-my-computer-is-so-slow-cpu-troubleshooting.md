---
layout: post
title: "Lab 07 – \"My Computer Is So Slow\": CPU Troubleshooting with Task Manager"
date: 2026-11-04
---

Goal
Practise one of the most common issue of a slow computer by deliberately overloading the CPU on a Windows 11 VM, finding the culprit in Task Manager, 

fixing it, and checking startup apps to stop slowness coming back.

Environment
- Windows 11 Home VM in VirtualBox
- Tools: Task Manager (Performance, Processes and Startup apps tabs), PowerShell

The analogy: a restaurant kitchen
- CPU is the chef. It does the actual work.
- RAM is the bench space. It holds whatever the chef is working on right now.
- Disk is the pantry. Fetching from there is slower than grabbing something off the bench.
- Network is the delivery door. Sometimes "slow" really means waiting on a delivery.

When someone says "it's slow", the job is working out which part of the kitchen is overloaded.

Part 1 – Baseline (what "good" looks like)

Opened Task Manager with Ctrl + Shift + Esc and went to the Performance tab.

[Screenshot: Performance tab at baseline]

- CPU: about 0.1%
- Memory: about 43%

What happens when memory fills up?
When the bench gets nearly full, Windows starts moving things it hasn't used in a while onto the page file, which lives on the disk. This is called paging. Things get sluggish but still respond. When RAM is completely full, apps start freezing or showing "Not Responding", because everything is being shuffled between the bench and the much slower pantry shelf. A useful clue on a call is high memory plus high disk activity when the user isn't opening or saving anything.

Part 2 – Breaking it

Opened PowerShell (not admin) and ran an endless loop:

    while ($true) {}

[Screenshot: PowerShell running the loop]

With one loop running, CPU only rose to about 35% and the VM felt fine.

Why only 35%? Each loop keeps one CPU core busy. Total CPU is the average across all cores, so one core flat out with the others idle shows up as roughly a third. Switching the graph to Logical processors (right-click the graph > Change graph to > Logical processors) showed one core pinned near 100%.

Help desk gotcha: a user can say one program is very slow while total CPU looks fine, because that program is maxing out a single core.

I then ran the loop in three PowerShell windows, one per core. CPU sat at 97–99%.

[Screenshot: CPU at 99%]

Surprisingly, the VM was still usable. File Explorer and Calculator opened fine, just a bit slower. Two reasons:
1. Windows gives a priority boost to whatever the user is actively using, so foreground apps jump the queue ahead of background work.
2. Only the CPU was overloaded. Memory stayed at 43% and the disk was quiet. Truly unusable PCs are usually a combination of high CPU, nearly full memory and a busy disk.

Lesson: 99% CPU on its own doesn't always mean an unusable PC. The numbers tell you where to look, and how the machine actually feels tells you how urgent it is.

Part 3 – Troubleshooting it like a ticket

In the Processes tab I sorted by the CPU column. Windows PowerShell was at the top, with three entries at about 30% each, adding up to the 99%. Memory at 43% confirmed it was a CPU problem, not a memory problem.

[Screenshot: Processes tab sorted by CPU]

Before ending anything on a real user's PC:
- Find out what the process is first. The top process could be an antivirus scan, Windows Update, or a system process Windows needs.
- Right-click > Open file location. A process running from the Windows or Program Files folders is very different from one running out of Downloads or a temp folder.
- Right-click > Search online for anything unfamiliar.
- Ask the user to save their work before closing any of their programs.

Fix: right-clicked the Windows PowerShell group and chose End task.

Verification: CPU dropped straight back to baseline, a cliff drop on the graph. On a real ticket, I'd also confirm with the user that the PC feels normal before closing the ticket.

[Screenshot: CPU back to baseline]

From experience: when a desktop is completely frozen, Ctrl + Alt + Delete into Task Manager is often the only way to kill a hung program. Windows handles that key combination at a lower level, so a frozen app can't block it.

Part 3b – Startup apps

Ending a task fixes the problem now, but the program can come back after a restart. I checked the Startup apps tab to see what launches at boot.

[Screenshot: Startup apps tab]

Found: Microsoft Edge, OneDrive, VirtualBox Guest Additions (low impact) and Windows Security notification (medium impact).

End task vs Disable:
- End task stops a running process right now. It's temporary.
- Disable in Startup apps stops a program launching automatically at the next boot. It doesn't stop it if it's already running, and it doesn't uninstall it. The user can still open it themselves.

If a PC takes a long time to boot or log in, lots of enabled High-impact startup apps (for example Spotify or Teams) are a common cause.

Judgment calls:
- Leave security software, drivers and tools like VirtualBox Guest Additions enabled.
- Ask the user before disabling something they rely on. Some people want Outlook open the moment they log in.

Lessons learned
- Total CPU is an average across cores, so one maxed-out core can hide behind a low total.
- High CPU alone doesn't always make a PC unusable. Check CPU, memory and disk together.
- Identify a process before ending it, especially on someone else's machine.
- End task is a fix for now. Startup apps is how you stop slowness coming back after a restart.
- Always verify the fix and confirm with the user.

Skills shown
- Task Manager: Performance, Processes and Startup apps tabs
- Per-core CPU monitoring
- Identifying and safely ending a resource-hungry process
- Managing startup programs to improve boot time
- Structured troubleshooting: baseline, break, diagnose, fix, verify

Next steps
- Resource Monitor (resmon) for a deeper look at which processes are using disk and network
- A memory pressure version of this lab
