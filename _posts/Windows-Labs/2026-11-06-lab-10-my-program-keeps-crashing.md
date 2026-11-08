---
layout: post
title: "Lab 10: \"My Program Keeps Crashing\" – Using Reliability Monitor"
date: 2026-11-06
---

6 November 2026

Goal
Use Reliability Monitor to find out what has gone wrong on a Windows 11 PC and when, then dig deeper in Event Viewer and Windows Update history to work out whether a problem needs fixing.

Environment
Windows 11 VM in VirtualBox (Win11-Lab). Tools used: Reliability Monitor, Event Viewer, Settings > Windows Update > Update history.

Background
When a user says "my program keeps crashing", Event Viewer has all the answers, but it is like scrolling through a whole building's security camera footage. Reliability Monitor is the manager's daily logbook: one column per day, with a red X on the days something broke and a short note on what. You start with the logbook to find the bad day, then pull the footage for just that day.

Reliability Monitor gives the PC a stability score from 1 to 10 and sorts events into rows:
- Application failures
- Windows failures
- Miscellaneous failures
- Warnings
- Information (often updates installed)

Red X = failure, yellow triangle = warning, blue i = information.

Before touching anything: questions to ask the user
If someone rang and said "Outlook keeps crashing", I would ask:
- How long has it been happening, and how many times?
- Is it just you, or is everyone having the problem? (One person points to their PC or profile. Everyone points to a server, an update or the service itself.)
- What are you doing when it crashes? Opening Outlook, sending an email, sending something big? Crashing on large emails can point to a bloated PST file or mailbox.
- What else is running at the same time?
- Has anything changed recently? A new add-in, an update, new software?
- When did it last happen, roughly, and did you see an error message?

The "when" question matters because Reliability Monitor is sorted by day, so a rough time takes you straight to the right column.

Step 1: Take a snapshot
In VirtualBox I took a snapshot called "Before Lab 10" so I could roll back if needed.

Step 2: Open Reliability Monitor
Start > type reliability > View reliability history.
Shortcut: Run box > perfmon /rel

[Screenshot: Reliability Monitor graph]

Step 3: Read the graph
My VM showed two red X's under Windows failures over the last few days, both saying "Windows was not shut down properly". Most of the rest were blue information icons for successful Windows updates, plus one yellow warning for a failed update.

Finding 1: "Windows was not shut down properly"
This message means Windows never got to run its shutdown. The power was cut from underneath it, like flicking off the main switch at the fuse box while the kitchen is still running. Nothing breaks, but there is tidying up to do on the next start.

Leaving an app like Notepad open does not cause this. A normal Start > Shut down asks every app to close, closes any stragglers itself and still counts as a clean shutdown.

The cause in my case: when I finished my last labs, I closed the VirtualBox window and powered the VM off, instead of shutting down Windows inside it first. To the VM, that is the same as pulling the plug out of the wall.

VirtualBox gives three options when you close a VM:
1. Save the machine state – freezes it, like pausing a film. Clean.
2. Send the shutdown signal – the same as Start > Shut down inside Windows. Clean.
3. Power off the machine – pulling the plug. This is what leaves the red X.

New habit: shut down Windows inside the VM first, or choose "Send the shutdown signal".

This was a lesson in VMs as much as Windows. On a normal laptop there is only one operating system to shut down. With a VM there is a guest (Windows 11 inside) and a host (my laptop running VirtualBox), and each has its own shutdown.

On a real PC, repeated "not shut down properly" messages are worth investigating. They can mean blue screens, a failing power supply, overheating or a dying laptop battery. The matching Event Viewer entries are Event ID 41 (Kernel-Power) and Event ID 6008.

[Screenshot: Windows failure details in Reliability Monitor]

Finding 2: Failed Windows update
The failed update was a yellow warning, not a red failure. Windows is saying "this didn't work, but nothing is broken". Windows usually retries on its own.

Reliability Monitor's summary only said the update failed, so I went one level deeper.

Settings > Windows Update > Update history lists each update with its KB number. Two updates had been installed, on the 2nd and the 4th.

Event Viewer > Applications and Services Logs > Microsoft > Windows > WindowsUpdateClient > Operational is the dedicated update log. Event ID 20 means an install failed. I had no Event ID 20. Instead I found Event ID 31: Windows failed to download an update.

[Screenshot: Event ID 31 in the WindowsUpdateClient Operational log]

My first theory was that I took a snapshot while the update was still running. But a live snapshot only pauses the VM for a second or two and Windows doesn't notice. Restoring an old snapshot mid-update, or powering off the VM mid-download, are more likely to cause problems.

Event ID 31 is a download failure, not an install failure. The update never arrived in one piece, so there was nothing to install. Common causes are a network blip, the VM being paused or closed mid-download, or Microsoft's servers being busy. The delivery truck didn't make it, so it came back the next day.

The updates on the following days all installed successfully, so this was a transient error: it happened once and fixed itself.

What I would do on a real ticket
In this case, nothing needs fixing. Later updates succeeded, so the update system is working. While I was at the PC I would:
1. Run Check for updates.
2. Restart.
3. Check Event Viewer for new errors.
4. Write up what I found in the ticket, even though no fix was needed, then close it.

When I would act: if the same KB kept failing, especially with the same error code. That is a pattern, not a blip. Next steps would be:
- Check free disk space
- Run the Windows Update troubleshooter
- Restart the Windows Update service
- Escalate with the KB number and error code in the ticket

Workplace note: in many organisations, updates are managed centrally (for example Intune or WSUS), and IT sometimes holds an update back on purpose. On a work PC, check the team's update policy before manually installing updates. For a small business with no central system, checking for updates on the spot is fine.

What I learned
- Reliability Monitor is the fastest first look when a user says something "keeps crashing". Start with the logbook, then pull the footage.
- Scoping questions before opening any tool save time: is it just you, when did it start, and what changed?
- A red X isn't always a crisis. Match it to what was happening that day, and the cause may be innocent.
- Warnings are not failures. Check whether things succeeded afterwards before acting.
- Use the dedicated logs: WindowsUpdateClient Operational is far quicker than searching the whole System log.
- Useful Event IDs: 41 and 6008 (unexpected shutdown), 20 (update install failed), 31 (update download failed).
- In a VM, always shut down the guest OS before closing VirtualBox.
