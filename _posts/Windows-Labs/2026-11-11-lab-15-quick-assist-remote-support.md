Lab 15: Quick Assist Remote Support

The ticket

"My Excel keeps doing something weird and I can't explain it over the phone. Can you just look?"

The help desk asks the user to share their screen so the technician can see what they see and, with their permission, take control.

How Quick Assist works

Quick Assist is like giving a tradie a one-time gate code.
- The helper (the technician) signs in with a Microsoft account and gets a 6-digit code. That's the gate code. It expires after 10 minutes.
- The user (the sharer) types the code in, sees who is asking to connect, and decides whether to allow it.
- The user can watch everything, pause the session, or end it at any time. It's their house, and they can walk you out the gate.

The security side

Scammers love Quick Assist because it's built into Windows and looks legitimate. A fake "Microsoft support" caller talks someone into typing in a code, and then they have control of the PC. Ransomware groups have used exactly this trick.

The golden rule for users: only enter a code when you contacted IT yourself, never because someone rang you out of the blue. Many organisations block Quick Assist and use a managed tool instead, such as Intune Remote Help.

Lab setup

- Helper: my host laptop, signed in with a lab-only Microsoft account, so my real account stays out of it.
- User: my Win11-Ent-Lab VM. The person sharing their screen doesn't need to sign in.

Part 1: Getting ready

1. Took a VirtualBox snapshot of the VM called "Before Lab 15".
2. Checked that Quick Assist was on both machines (Start, type Quick Assist). It was already installed on both.

Part 2: Making the connection

On the host (helper): opened Quick Assist, clicked Help someone, signed in with my lab account and got a 6-digit code with a 10-minute countdown.

On the VM (user): opened Quick Assist, typed the code under Get help and clicked Submit. The next screen showed the name of the person asking to connect. This is the screen where a user should stop and check it's really their IT person before clicking Allow.

Back on the host the VM's screen appeared, view only at first. Clicking Request control sent a request to the VM, and once it was approved I could control the VM from the host.
[Screenshot: lab15-01-session-working.png]

Why big organisations often don't use Quick Assist

When I worked help desk at ANZ we didn't use Quick Assist. The bank had its own remote control software. Thinking about it now, that makes sense for a bank:
- Scammers know Quick Assist is on every Windows PC. If staff are used to IT connecting with Quick Assist, a scammer who rings pretending to be IT looks just like the real thing.
- Anyone with a free Microsoft account can be a Quick Assist helper. A managed tool only lets approved IT staff connect.
- Managed tools log every session (who connected, to which PC, when), which a bank needs for audits and investigations.
- IT can block Quick Assist completely, so a scam call has nothing to work with.

Part 3: The user's power

The user stays in charge for the whole session.

Pause: when the user clicked Pause on the VM, the helper's view was blanked out with a blue screen. The helper can't see anything until the user clicks Resume. It's like the user drawing the curtains for a moment. A user would pause when they need to type a password or open something private, like their banking or personal email, so the technician never sees it.

Ending the session: when the user clicked Leave on the VM, the helper's window didn't simply close. It showed a message saying screen sharing had ended. That tells the technician the user chose to end it, rather than the connection dropping. The user can do this at any moment, and the helper can't stop them or reconnect without a new code.

Help desk checklist for a remote support call

1. Confirm who the user is (follow the organisation's identity check).
2. Explain what you're going to do and that they stay in control.
3. Give them the code yourself over the phone or ticket. Never ask them to read a code to you.
4. Tell them to check the name on the consent screen before clicking Allow.
5. Start with view only. Request control only if you need it.
6. Ask them to pause whenever they type a password or open anything private.
7. Fix the problem, explain what you did, and let them end the session.
8. Note in the ticket that a remote session was used and what was changed.

What I learned

- Quick Assist lets a helper see, and with permission control, another Windows PC using a 6-digit code that expires after 10 minutes.
- The user stays in charge: they see who is connecting, approve control, can pause to hide their screen, and can end the session at any time.
- Pause protects both sides. The technician never sees passwords or private information.
- The consent screen with the helper's name is the user's main protection against scammers.
- Anyone with a free Microsoft account can be a helper, which is why scammers use it and why many organisations block it and use a managed tool with logging instead, like the remote control software I used at ANZ.
- The golden rule for users: only enter a code when you contacted IT, never because someone rang you.

Scam call scenario

The call: "Someone from Microsoft called me, said my computer had a virus, and I typed in a code they gave me. They were moving my mouse around. Then I got worried and hung up."

The Quick Assist session itself is over once the user hangs up and closes it, because the code only works once. The real worry is what the scammer did while they were in. A common trick is to quickly install a second remote access tool like AnyDesk, so they can get back in later without the user's help. Users often don't realise what remote access means until it's too late.

What I would do, in order:

1. Stay calm and don't blame the user. People who feel they're in trouble leave things out. Let them know they did the right thing by calling, then ask honest questions: how long were they connected (5 minutes or 20?), what did the scammer do, and what was open on the screen (email, banking, customer data)?
2. Isolate the computer. Unplug the network cable or turn off Wi-Fi, but leave the computer switched on. Turning it off can wipe evidence held in memory that the security team may need.
3. Escalate to security straight away. This is now a security incident, not just a help desk ticket. The security team can check whether the attacker moved from that PC to anything else on the network.
4. From a different, clean computer, reset the user's passwords, sign them out of all sessions and revoke active sessions (email, Microsoft 365, VPN). Never change passwords on the PC that was accessed, in case a keylogger was installed.
5. With security's okay, check the PC for new remote access software (AnyDesk, TeamViewer and similar), new user accounts, startup items and unusual processes, and run scans.
6. Wipe and rebuild the PC once security has what they need. With an unknown attacker you can't be sure what's hiding on it, so a clean rebuild is the safe choice, especially for users with high-level access.
7. Document everything: date and time, computer name, username, how long the scammer was connected, what was on screen and what was done. Then the team monitoring the network knows what to look out for.

Notes on reporting

- VirusTotal is a tool for checking suspicious files, hashes and links, not a place to report incidents. Check a file's hash first rather than uploading it, because uploaded files can be seen by other VirusTotal subscribers, and you don't want to leak company data.
- MITRE ATT&CK is a library describing attacker techniques. It's useful for understanding and describing what happened, but you don't report to it.
- In Australia, cyber incidents can be reported through ReportCyber (cyber.gov.au), and scams through Scamwatch. In a company, the security team or management decides on external reporting, not the help desk.

Help desk lesson: on the first call, my job is to contain the problem, escalate it and write it down, not to investigate it alone.
