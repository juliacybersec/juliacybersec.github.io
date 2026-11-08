---
layout: post
title: "Lab 08: \"My Printer Won't Print\" – Clearing a Stuck Print Queue"
date: 2026-11-06
---

6 November 2026

Goal
"My printer won't print" is one of the most common help desk calls. In this lab I jammed the print queue on my Windows 11 VM on purpose, then worked through it the way I would on a real ticket: gather information, look at the queue, fix it, and confirm it's actually fixed before closing the ticket.

Environment
- Windows 11 Home VM in VirtualBox
- Microsoft Print to PDF (the printer built into Windows 11, so no real printer needed)
- Snapshot taken first: "Before Lab 08"

[Screenshot: VirtualBox snapshot list showing Before Lab 08]

My analogy
The print spooler is the queue at the post office. Your documents line up and get handed to the printer one at a time. If the counter is closed, or one parcel gets stuck, everyone behind it waits, even if the printer itself is fine.

Before touching anything: gather information
When someone rings up and says "my printer won't print", I don't jump straight in. In a big office like the bank I worked at, there can be several printers on every floor, so the first questions are:
- Which printer are you printing to? (People often think they know and don't, but the queue will tell me.)
- What are you trying to print? If it's a Word doc, I can look for it in the queue.
- Is there anything showing on the printer itself? Error message, paper, toner?
- Can anyone else print to it? If the whole floor can't print, it's the printer or print server. If it's just this one person, it's their PC: their queue, their driver, or their default printer.
- Is it set as their default printer? A lot of these calls are jobs quietly going to the wrong printer.

Breaking it on purpose
My plan was to tick Pause Printing in the queue, but that option wasn't available for Microsoft Print to PDF. So I found another way: in Printer properties, Advanced tab, I changed the printer from "Always available" to only available from 1am to 3am, then clicked Apply and OK.

[Screenshot: Printer properties, Advanced tab, availability set to 1am–3am]

This turns out to be a real-world cause too. Some offices limit when big printers can be used, and if the hours get set wrong, jobs just sit there with no obvious error.

I then printed three test pages from Notepad (test print 1, 2 and 3) and opened the print queue.

What the queue showed
All three jobs were sitting in the queue with no error, just an unknown status. That's a clue in itself. When jobs are waiting with no error, the jobs are usually fine and the problem is the printer or its settings. When a single job at the top shows "Error", that's the stuck parcel and you delete that one job.

[Screenshot: print queue with three jobs waiting]

Fix attempt 1: change the setting back
I set the printer back to "Always available". The jobs stayed stuck. I tried Resume All, refreshing, and pausing and resuming. Still stuck.

My theory: Windows was following the rules it had when the jobs arrived. They were accepted under "closed until 1am", and the spooler hadn't looked at the new setting since. Cancelling the jobs and sending them again would have worked, and on a real ticket that's often what you'd do.

Fix attempt 2: restart the Print Spooler
Instead, I used the classic help desk fix: restarting the Print Spooler service. (This links back to my svchost lab, because the spooler is a Windows service.)

1. Windows key + R, type services.msc, press Enter
2. Scroll down to Print Spooler
3. Right-click and choose Restart

[Screenshot: Services window with Print Spooler selected]

Back in the queue, each job now showed "Error - Printing" with a red mark. That looks scary to a user, but it's just because the spooler restarted underneath the waiting jobs. I clicked Resume All and all three printed straight away.

Analogy: the post office worker read the "closed until 1am" sign when the parcels came in and never looked up again. Restarting the spooler puts a fresh worker on shift, who reads the sign as it is now.

[Screenshot: queue showing Error - Printing before Resume All]

Checking it actually worked
An empty queue doesn't prove anything printed. I checked my Documents folder and all three PDFs were there.

[Screenshot: Documents folder with the three test PDFs]

On a real ticket, I'd ask the user to print something they actually need rather than a test page, so no paper is wasted and we both know it's working before I close the ticket. The last thing you want is a call back five minutes later.

What I learned
- Gather information first: which printer, what job, any errors on the printer, and whether anyone else can print.
- The queue tells you a lot. Jobs waiting with no error usually means a printer or settings problem. One job showing an error means delete that job.
- Changing a setting doesn't always fix jobs already in the queue. Restarting the Print Spooler makes Windows look again.
- After a spooler restart, jobs may show an error and need Resume All. That's an easy step to forget, and the user rings back saying it still isn't printing.
- Always confirm the fix before closing the ticket.
