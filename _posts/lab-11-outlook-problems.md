Lab 11: Outlook Problems (New Outlook on Windows 11)

Goal

Practise the most common help desk email tickets: mail not arriving, "missing" emails, a broken Outlook app and sign-in problems after a password change. I also worked out which troubleshooting steps apply to new Outlook and which only apply to classic Outlook.

Lab setup

- Windows 11 VM in VirtualBox (Win11-Lab)
- New Outlook (Outlook for Windows), built into Windows 11
- A lab-only outlook.com account created under a pseudonym, so nothing I broke, rebuilt or screenshotted touched my real email

How I think about Outlook

I picture Outlook as a post office counter at home:

- The mailbox (Exchange / Microsoft 365 / outlook.com) is the real post office in town. That is where the mail actually lives.
- The Outlook app is the counter at home, where you read and send mail.
- Add-ins are extra helpers standing at the counter.
- In classic Outlook, the OST file is a photocopy of the mailbox kept on the counter so you can read it fast, even offline.

Most Outlook tickets come down to working out which of these has broken.

Step 1: Identify which Outlook the user has

Windows 11 now has two Outlooks, and the fixes are different, so this is always the first question.

- The app opened on a "Welcome to the new Outlook" screen
- There was no Mail item in Control Panel (this only appears with classic Outlook, because it is where classic profiles are managed)
- Settings > Apps > Installed apps showed Outlook for Windows, not Microsoft 365 or Office

Result: new Outlook only.

[Screenshot: Welcome to the new Outlook screen]

[Screenshot: Installed apps showing Outlook for Windows]

Side ticket 1: account creation blocked

Creating the outlook.com account inside the VM was blocked as suspicious activity. A brand new virtual machine with a fresh browser and no history can look like a bot. Creating the account on my normal laptop worked, and signing in on the VM afterwards was fine.

Help desk lesson: "I can't create or sign in to my account, it says suspicious activity" is a real ticket. The fix is to change the device or network that looks risky, wait out the lockout, or verify identity. Never try to trick the security check.

I also learned that an outlook.com address always comes with a Microsoft account. The account is the identity (the login) and the mailbox is attached to it. At work it is the same idea: when someone says "my email won't work", sometimes it is the account (password, MFA, lockout) and sometimes it is the mailbox.

Side ticket 2: host disk full

Partway through, VirtualBox paused the VM with a VERR_DISK_FULL error. The full disk was my actual laptop, not the VM. The VM's hard drive is one big file on the laptop that grows as it is used, and snapshots add more files. I left the VM paused (powering off mid-write can corrupt the disk), freed space on the laptop, then resumed it.

Side ticket 3: the VM kept shutting down

Later the VM switched itself off just as I opened Outlook. Instead of rolling back to a snapshot, I tried to find the cause first, the way you would on a real ticket.

1. VirtualBox log: the last lines showed my network test and then nothing. No error, no clean power-off line. The log just stopped.
2. Event Viewer inside the VM: Event ID 41 (Kernel-Power) and Event ID 6008 (unexpected shutdown). These are written when Windows starts back up and notices it was not shut down properly.
3. BugcheckCode on the Event 41: 0. That means there was no blue screen, so Windows did not crash. Something outside the VM pulled the plug.
4. Event Viewer on the laptop: a DCOM 10016 warning, which is harmless and appears on almost every Windows machine (knowing which warnings to ignore is a skill in itself).
5. VM display settings: 3D acceleration off, 128 MB video memory, VBoxSVGA controller. All fine, so ruled out.
6. VM memory: 8 GB, which is half of my laptop's RAM. The laptop was running very close to full.

Change made: base memory 8192 MB to 4096 MB, one change at a time so I would know what fixed it. Retest: opened Outlook. The VM stayed up for the rest of the lab.

[Screenshot: Event 41 with BugcheckCode 0]

[Screenshot: VM base memory setting]

Test 1: the PC is offline

1. Confirmed the mailbox was working in Outlook on the web. This is the baseline: if mail works on the web, the post office is fine and the problem is on that computer.
2. Disconnected the VM's network adapter (like pulling out the network cable).
3. Tried to send an email from the VM. Outlook showed it was offline and the email waited in the Outbox.
4. Reconnected the network. The emails in the Outbox sent automatically. No need to send them again.

[Screenshot: Outlook showing offline]

[Screenshot: email waiting in the Outbox]

[Screenshot: Outbox empty after reconnecting]

Help desk tip: when a user says an email they sent "disappeared", check the Outbox first. If it is stuck there while they are online, that points to a sign-in, sync or attachment-size problem instead.

Other things to check when mail works on the web but not in the app:

- Is the PC online, or is Outlook set to work offline?
- Is the app signed in to the right account?
- Is the mail in the Other tab, Junk or another folder?
- Has the sign-in expired?
- Does the app need restarting, updating, repairing or resetting?
- Is a firewall, VPN or proxy blocking it?
- Is the Windows date and time wrong?

Go quickest and least disruptive first: ask, look and restart before resetting anything.

Test 2: "My invoice email never arrived"

1. Set a trap: a rule moving any email with "invoice" in the subject to Archive.
2. Sent an email called "Invoice test" from an outside address.
3. It was missing from the inbox in both the VM app and Outlook on the web.

This was the twist. Missing in both places does not always mean the mailbox is broken. Rules in outlook.com and Microsoft 365 run on the server, before the mail reaches any app, so the web-versus-app test cannot catch them.

How I found it: searched for "invoice". Search checks every folder at once and showed the email was in Archive. Folders to check for mail someone is expecting to receive: Junk, the Other tab, Archive and Deleted Items. Also ask whether the sender got a bounce-back.

Finding the email fixes the symptom, not the cause. To stop it happening again, check the settings that can move mail automatically:

- Rules
- Junk email settings
- Sweep (an outlook.com feature that moves or deletes mail from certain senders)
- Focused Inbox

I deleted the rule, sent "Invoice test 2", and it landed in the inbox. Sometimes you need to click off the folder and back on, or wait a few seconds, for it to show.

[Screenshot: search results showing the email in Archive]

[Screenshot: the rule in Settings > Mail > Rules]

[Screenshot: Invoice test 2 in the inbox]

Security link: a rule the user did not create is a red flag for a compromised account. In business email compromise (BEC), an attacker logs into a mailbox and adds a hidden rule that moves emails about invoices or payments out of sight, so the real owner never sees them while the attacker redirects the payment. If a user says they did not make the rule, escalate it to security, reset their password, revoke their sessions and check their sign-in history.

Test 3: Repair and Reset

Settings > Apps > Installed apps > Outlook > Advanced options.

- Repair is like tidying the counter. The app fixes its own files and keeps the account and settings. After Repair, I was still signed in and all my emails were there. (Microsoft also sent a welcome email, probably because the refreshed app looked like a new sign-in.)
- Reset is like clearing the counter completely. It wipes the app's data on that computer, including the sign-in. My emails were all still there, because they live on the server, not in the app.

Rule of thumb: Repair first, Reset second. Before a Reset, check the user knows their password and has their MFA method handy, or an "Outlook is broken" ticket becomes a "locked out" ticket.

[Screenshot: Advanced options page with Repair and Reset]

[Screenshot: Outlook after Reset]

Test 4: "I changed my password and now Outlook keeps asking me to sign in"

1. Changed the lab account's password on my laptop.
2. Sent a test email. It still arrived in Outlook on the VM.
3. Signed out of the account in Outlook and signed back in. The old password was refused. The new one worked and everything synced back.

Why it kept working at first: modern Outlook does not keep using the password. When you sign in, the app gets a token, like a visitor's pass, and shows it each time it syncs. Changing the password does not always cancel passes that were already handed out, so the app keeps working until the pass expires or is cancelled. That is why a user can change their password in the morning and only get sign-in prompts after lunch.

Security link: changing the password alone does not always kick out an attacker who has stolen a session or token. On a compromised account, reset the password and revoke all sessions so everyone has to sign in again.

[Screenshot: sign-in screen refusing the old password]

Classic Outlook fixes (theory)

These do not apply to new Outlook. I tried outlook.exe /safe on the VM and it is not supported, because safe mode is a classic Outlook feature. For new Outlook, the equivalent is Repair and then Reset. Most workplaces still use classic Outlook, so these are worth knowing:

- Safe mode (outlook.exe /safe): starts Outlook without add-ins. If it works in safe mode, an add-in is the problem. Turn add-ins off one at a time in File > Options > Add-ins to find it.
- Rebuild the OST: the OST is a synced copy of the mailbox on the PC. Close Outlook, rename the .ost file (safer than deleting, because you can undo it) and let Outlook download a fresh copy. Check the Outbox first, because anything not yet synced can be lost.
- New profile: Control Panel > Mail > Show Profiles > Add. This is the Outlook version of the Windows profile fix I used on the help desk at ANZ: start clean and sign back in.
- PST files are different. A PST is a personal file that exists only on that PC, used for POP3 accounts and for archives of old mail. There is no copy on the server, so deleting a PST deletes the mail unless there is a backup. A corrupted PST is repaired with the Inbox Repair Tool (scanpst.exe), after backing it up first.
- Password prompt loops: clear old saved logins in Credential Manager > Windows Credentials, then sign in again.

What I learned

- The first question on any Outlook ticket is which Outlook the user has, because the fixes are different.
- Outlook on the web is the quickest test of whether the problem is the mailbox or the computer.
- Missing in both web and app can still be a rule or filter on the server, not a broken mailbox.
- Fix the cause, not just the symptom.
- Change one thing at a time and write down what you changed.
- Find the root cause before reaching for a big fix like a rollback.
- Some help desk problems are security problems in disguise: an unexpected rule or a session that will not end can mean a compromised account.
