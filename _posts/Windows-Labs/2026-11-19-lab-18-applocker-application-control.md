---
title: "Lab 18: AppLocker – Essential Eight Application Control on a Domain Workstation"
date: 2026-11-19
---

Lab 18: AppLocker – Essential Eight Application Control on a Domain Workstation

Lab date: 19 November 2026
Environment: VirtualBox on my laptop – Windows Server 2025 domain controller (DC01) and a Windows 11 Enterprise workstation (Win11-Ent-Lab), both in my silkylab.local domain on an isolated NAT network


What I Built

I deployed application control to a domain-joined Windows 11 workstation using AppLocker and Group Policy, following the staged rollout I'd use in a real environment:

- Moved the workstation out of the default Computers container into a dedicated Lab-Workstations OU, so computer policy could be targeted at it
- Built a dedicated GPO, Lab-AppLocker, that sets the Application Identity service to start automatically and creates the default allow rules for executables, Windows Installer files, scripts and packaged apps
- Ran the policy in audit-only mode first and read the AppLocker event logs to see what would have been blocked
- Found and fixed a misconfiguration during testing: one rule collection had been set to Enforce instead of Audit only
- Switched all four rule collections to Enforce and verified the policy's decisions with Test-AppLockerPolicy, without running the test files
- Documented a deliberate gap against the Essential Eight: DLL rules were left off

[Screenshot: lab18-04b-gpo-scope-linked.png]


Why It Matters

Application control is one of the Australian Signals Directorate's Essential Eight mitigation strategies, and one of the most effective. Most malware arrives as something a user downloads or opens from their profile folders. If only approved programs can run, a malicious .exe or script dropped in Downloads simply doesn't execute.

The control itself is simple. The skill is in rolling it out without stopping a business from working: audit first, read the logs, fix the rules, then enforce. This is the kind of control I would expect to assess for a mid-size client, measuring where they sit against the Essential Eight maturity levels and showing them the gaps with evidence from their own logs.


Lab Setup

- DC01: Windows Server 2025, domain controller for silkylab.local (built in Lab 16)
- Win11-Ent-Lab: Windows 11 Enterprise evaluation, joined to silkylab.local. AppLocker enforcement needs Enterprise or Education.
- Test user: a standard domain user account with no admin rights (from Lab 16)
- VirtualBox Guest Additions installed on Win11-Ent-Lab, with Shared Clipboard set to Bidirectional, so long commands can be pasted in rather than retyped

Note: I've left out IP addresses, passwords and the workstation's real computer name on purpose. DESKTOP-XXXXXXX stands in for the computer name.


Steps (summary)

Every command is also on my Command Cheat Sheet and Commands Explained pages.

Part A – Give the workstation its own OU

AppLocker is a computer policy, so the GPO must be linked to an OU that contains the computer. GPOs can't be linked to the default Computers container, and keeping workstations in their own OU is standard practice anyway.

1. [DC01 – SILKYLAB\Administrator] Found the workstation's real computer name in AD:

```
Get-ADComputer -Filter * | Select Name
```

   What it does: lists every computer account in Active Directory and shows only their names.
   - Get-ADComputer: the cmdlet that reads computer accounts from Active Directory
   - -Filter: which computers to return
   - `*`: a wildcard meaning all of them
   - |: the pipe, which hands the results to the next command
   - Select: short for Select-Object, keeps only the columns named
   - Name: the computer's name

   Finding: the computer name in AD was the random name Windows gave it during setup, not the VM name I'd used in VirtualBox. VirtualBox and Windows keep separate names. In production, a naming convention avoids this, because random names make inventory and log searches harder.

[Screenshot: lab18-01-get-adcomputer.png]

2. [DC01 – SILKYLAB\Administrator] Created the OU:

```
New-ADOrganizationalUnit -Name "Lab-Workstations" -Path "DC=silkylab,DC=local"
```

   What it does: creates a new OU called Lab-Workstations at the top level of the domain.
   - New-ADOrganizationalUnit: the cmdlet that creates an Organizational Unit
   - -Name: the new OU's name
   - "Lab-Workstations": the name, in quotes so it's read as one piece of text
   - -Path: where in AD to create it
   - "DC=silkylab,DC=local": the domain root as a distinguished name. DC means Domain Component, and the comma separates the two parts of silkylab.local

3. [DC01 – SILKYLAB\Administrator] Saved the computer account in a variable:

```
$pc = Get-ADComputer "DESKTOP-XXXXXXX"
```

   What it does: finds the workstation's computer account and stores it so the next command stays short.
   - $pc: a variable named pc. The $ marks it as a variable
   - =: puts the result on the right into the variable on the left
   - Get-ADComputer: reads a computer account from AD
   - "DESKTOP-XXXXXXX": the computer's name, in quotes

4. [DC01 – SILKYLAB\Administrator] Moved it into the new OU:

```
Move-ADObject $pc -TargetPath "OU=Lab-Workstations,DC=silkylab,DC=local"
```

   What it does: moves the workstation out of the Computers container and into Lab-Workstations.
   - Move-ADObject: the cmdlet that moves any AD object (user, computer, group)
   - $pc: the computer account saved in Step 3
   - -TargetPath: where to move it to
   - "OU=Lab-Workstations,DC=silkylab,DC=local": the destination as a distinguished name. OU= is the Organizational Unit, the DC= parts are the domain

5. [DC01 – SILKYLAB\Administrator] Confirmed the move:

```
(Get-ADComputer "DESKTOP-XXXXXXX").DistinguishedName
```

   What it does: shows the computer's full location in AD.
   - ( ): run the command inside first, then use its result
   - Get-ADComputer: reads the computer account
   - "DESKTOP-XXXXXXX": the computer's name
   - .: picks one property out of the result
   - DistinguishedName: the object's full path in AD

   Result: the path now included OU=Lab-Workstations instead of CN=Computers.

[Screenshot: lab18-02-computer-moved-to-ou.png]

Part B – Build the AppLocker GPO in audit mode

1. [DC01 – SILKYLAB\Administrator] In Group Policy Management (gpmc.msc), created a new GPO called Lab-AppLocker and linked it directly to the Lab-Workstations OU. As in Lab 17, I kept the change in its own clearly named GPO rather than editing a default policy, so it's easy to review, report on or roll back.

2. [DC01 – SILKYLAB\Administrator] Under Computer Configuration > Policies > Windows Settings > Security Settings > System Services, set Application Identity to Automatic. AppLocker rules do nothing unless this service is running, and setting it through the GPO means it can't be quietly left off on one machine.

3. [DC01 – SILKYLAB\Administrator] Under Security Settings > Application Control Policies > AppLocker, created the default rules for all four rule collections: Executable, Windows Installer, Script and Packaged app. The default rules allow anything in the Windows and Program Files folders for everyone, and allow administrators to run anything.

[Screenshot: lab18-03-default-rules-exe.png]

4. [DC01 – SILKYLAB\Administrator] In Configure rule enforcement, set all four collections to Audit only. I deliberately left DLL rules off (on the Advanced tab). See Key Learnings for why that is a documented gap rather than an oversight.

[Screenshot: lab18-04-enforcement-audit-only.png]

5. [DC01 – SILKYLAB\Administrator] Checked the GPO's Scope tab to confirm it was linked to Lab-Workstations.

Part C – Apply the policy and test audit mode

1. [Win11-Ent-Lab – SILKYLAB\Administrator] Pulled the new policy down:

```
gpupdate /force
```

   What it does: applies Group Policy now instead of waiting for the normal refresh (about every 90 minutes).
   - gpupdate: the tool that refreshes Group Policy
   - /force: reapply every policy, not just the ones that changed

2. [Win11-Ent-Lab – SILKYLAB\Administrator] Confirmed the GPO applied to the computer:

```
gpresult /r /scope:computer
```

   What it does: shows a summary of which computer policies applied to this PC.
   - gpresult: the tool that reports which Group Policy settings applied
   - /r: the short summary report
   - /scope:computer: computer policy only. The colon joins the switch to its value

   Result: Lab-AppLocker was listed under Applied Group Policy Objects.

[Screenshot: lab18-05-gpresult-applocker.png]

3. [Win11-Ent-Lab – SILKYLAB\Administrator] Checked the Application Identity service:

```
Get-Service AppIDSvc | Select Status, StartType
```

   What it does: shows whether the Application Identity service is running and how it starts.
   - Get-Service: the cmdlet that reads Windows services
   - AppIDSvc: the short name of the Application Identity service
   - |: hands the result to the next command
   - Select: keeps only the columns named
   - Status: running or stopped right now
   - ,: separates the two column names
   - StartType: Automatic, Manual or Disabled

   Result: StartType was Automatic, as set by the GPO.

[Screenshot: lab18-06-appidsvc-automatic.png]

4. [Win11-Ent-Lab – SILKYLAB\tuser] As the standard user, placed two test files in the Downloads folder, outside the folders the default rules allow: test-app.exe (a renamed copy of cmd.exe, standing in for an unapproved downloaded program) and hello.bat (a one-line batch script). Ran both.

5. [Win11-Ent-Lab – SILKYLAB\Administrator] Read the audit events for programs:

```
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-AppLocker/EXE and DLL';Id=8003} | Format-List TimeCreated,Message
```

   What it does: finds events showing a program was allowed to run but would have been blocked if AppLocker were enforced.
   - Get-WinEvent: the cmdlet that reads Windows event logs
   - -FilterHashtable: filters with a set of conditions inside the log itself, so it's fast
   - @{ }: a hashtable of name = value conditions. @ marks the start, the curly braces hold the conditions
   - LogName='Microsoft-Windows-AppLocker/EXE and DLL': the AppLocker log for programs and DLLs. Single quotes because the name has spaces
   - ;: separates the two conditions
   - Id=8003: "allowed, but would have been blocked" for programs
   - |: hands the events to the next command
   - Format-List: shows each event as a vertical list, so the message isn't cut off
   - TimeCreated,Message: when it happened and the full event text. The comma separates the two

   Result: an 8003 event for test-app.exe in tuser's Downloads folder. Audit mode was doing its job: the program ran, and the log recorded that enforcement would have stopped it.

[Screenshot: lab18-07-event-8003-exe.png]

6. Troubleshooting: hello.bat didn't run at all. The MSI and Script log showed an 8007 (blocked) event instead of the 8006 (audit) I expected. Reviewing the GPO, I found the Script rules collection had been set to Enforce rules rather than Audit only. One collection was enforcing while the other three were auditing. I corrected it to Audit only, ran gpupdate /force again, and re-tested.

[Screenshot: lab18-08-event-8007-script-blocked.png]

   This is exactly why a rollout starts in audit mode and why the enforcement settings get checked collection by collection. In a real organisation, that one misconfigured setting would have blocked every login script and admin batch file on every workstation in the OU.

7. [Win11-Ent-Lab – SILKYLAB\Administrator] Read the audit events for scripts after the fix:

```
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-AppLocker/MSI and Script';Id=8006} | Format-List TimeCreated,Message
```

   What it does: the same check for scripts and installers.
   - Same parts as Step 5, except:
   - 'Microsoft-Windows-AppLocker/MSI and Script': the AppLocker log for installers and scripts
   - Id=8006: "allowed, but would have been blocked" for scripts and installers

   Result: an 8006 event for hello.bat. The script ran and was logged.

[Screenshot: lab18-09-event-8006-script.png]

Part D – Enforce, and verify the policy's decisions

1. [DC01 – SILKYLAB\Administrator] In the Lab-AppLocker GPO, set all four rule collections (Executable, Windows Installer, Script and Packaged app) to Enforce rules. DLL rules were left off, as documented.

[Screenshot: lab18-10-enforcement-enforced.png]

2. [Win11-Ent-Lab – SILKYLAB\Administrator] Applied the change with gpupdate /force.

3. [Win11-Ent-Lab – SILKYLAB\Administrator] Created a fresh copy of the test program in a folder the administrator account could reach:

```
Copy-Item C:\Windows\System32\cmd.exe C:\Users\Public\test-app.exe
```

   What it does: copies Command Prompt into the Public folder under a new name, to stand in for an unapproved program.
   - Copy-Item: the cmdlet that copies a file (source first, then destination)
   - C:\Windows\System32\cmd.exe: the source, the real Command Prompt program
   - C:\Users\Public\test-app.exe: the destination. It's outside the Windows and Program Files folders, so the default rules don't allow it

   Troubleshooting: my first attempts at this part failed with a path-not-found error. I worked out that I was running the commands on DC01 instead of the workstation. The test user had never logged on to DC01, so the profile folder didn't exist there, and the AppLocker GPO doesn't apply to DC01 anyway, so any result there would have been meaningless. If you can't find a file in a user profile, this command finds the real path instead of guessing at folder names:

```
Get-ChildItem C:\Users -Recurse -Filter test-app.exe -ErrorAction SilentlyContinue | Select-Object FullName
```

   What it does: searches every user profile for a file by name and shows its full path.
   - Get-ChildItem: the cmdlet that lists files and folders
   - C:\Users: the folder to start searching in
   - -Recurse: also search every folder inside it
   - -Filter test-app.exe: only return items with this name
   - -ErrorAction SilentlyContinue: hide errors (such as folders you can't open) and keep going
   - |: hands the results to the next command
   - Select-Object FullName: show only the file's complete path

4. [Win11-Ent-Lab – SILKYLAB\Administrator] Asked AppLocker for its verdict on the unapproved program, without running it:

```
Get-AppLockerPolicy -Effective | Test-AppLockerPolicy -Path C:\Users\Public\test-app.exe -User SILKYLAB\tuser
```

   What it does: checks whether the standard user would be allowed to run the file under the rules in force on this PC, and which rule decides.
   - Get-AppLockerPolicy: the cmdlet that reads AppLocker rules
   - -Effective: the merged policy actually in force on this PC (GPO plus any local rules)
   - |: hands that policy to the next cmdlet
   - Test-AppLockerPolicy: checks a file against the policy and reports the decision
   - -Path: which file to test
   - C:\Users\Public\test-app.exe: the test program
   - -User: which user to test as
   - SILKYLAB\tuser: the domain, a backslash, then the username

   Result: PolicyDecision DeniedByDefault, with MatchingRule blank. No rule allows the file, so it is blocked automatically.

5. [Win11-Ent-Lab – SILKYLAB\Administrator] Ran the same test against an approved Windows program:

```
Get-AppLockerPolicy -Effective | Test-AppLockerPolicy -Path C:\Windows\System32\notepad.exe -User SILKYLAB\tuser
```

   What it does: the same check, for Notepad.
   - Same parts as Step 4, except:
   - C:\Windows\System32\notepad.exe: Notepad's location in the Windows folder

   Result: PolicyDecision Allowed, with MatchingRule showing the default rule for files in the Windows folder.

[Screenshot: lab18-11-test-applockerpolicy.png]

6. [Win11-Ent-Lab – SILKYLAB\Administrator] Removed the test copy:

```
Remove-Item C:\Users\Public\test-app.exe
```

   What it does: deletes the test program so nothing unapproved is left on the machine.
   - Remove-Item: the cmdlet that deletes a file or folder
   - C:\Users\Public\test-app.exe: the file to delete


Key Learnings

Allowlisting and implicit deny
I never wrote a rule to block test-app.exe. I only wrote rules to allow things. Once a rule collection has any rules and is enforced, anything not explicitly allowed is denied by default. That's the difference between application control and antivirus: antivirus tries to recognise bad files, application control only lets known-good ones run.

Audit first, then enforce
Audit mode records what enforcement would block (8003 and 8006) without stopping anyone working. In a real rollout you'd leave it in audit for a period, review the logs for legitimate business apps the default rules miss, add rules for them, and only then enforce. Going straight to Enforce risks blocking line-of-business apps, login scripts and installers across the whole OU. My accidentally enforced Script rules showed exactly that on a small scale.

AppLocker event IDs
- 8002 / 8005: allowed (programs / scripts and installers)
- 8003 / 8006: allowed, but would have been blocked (audit mode)
- 8004 / 8007: blocked (enforce mode)

Programs and DLLs log to the EXE and DLL log. Scripts and installers log to the MSI and Script log. These are the events a SOC analyst would see in a SIEM: a 8004 for an unknown .exe in a user's Downloads folder is worth a closer look.

Test-AppLockerPolicy
This lets an administrator ask "would this user be allowed to run this file, and which rule decides?" without running anything. It's useful for checking a policy before enforcing it, and for answering "why is this app blocked for one user but not another?" It must be run on a machine the policy applies to.

The DLL gap and the Essential Eight
The Essential Eight application control requirements at Maturity Level 1 cover software libraries (DLLs) as well as executables, scripts and installers. I left DLL rules off on purpose, because they check every library a program loads, which adds performance overhead and needs careful testing. In a real assessment this would be recorded as a gap with a remediation plan, not left silently undone. Honestly documenting a gap is part of the control.

AppLocker vs App Control for Business
Microsoft now recommends App Control for Business (formerly Windows Defender Application Control, WDAC) for new deployments. AppLocker is still supported but isn't getting new features. AppLocker is simpler and can apply different rules per user or group, which made it the right tool for learning the concepts here. App Control for Business applies to the whole device and is harder to bypass. Many organisations run both.

Which roles touch this control
- A sysadmin or endpoint engineer builds and maintains the policy, as in this lab.
- A SOC analyst never configures it but investigates its events in the SIEM.
- A GRC analyst or assessor checks the control exists, is enforced, and covers what the Essential Eight requires, and records any gaps.


Skills Demonstrated

- Structuring Active Directory with a dedicated workstation OU and targeting computer policy at it
- Designing and deploying an AppLocker application control policy through a dedicated GPO
- Running a staged rollout: default rules, audit mode, log review, then enforcement
- Reading and filtering the AppLocker event logs with Get-WinEvent and -FilterHashtable
- Verifying policy decisions with Get-AppLockerPolicy and Test-AppLockerPolicy
- Troubleshooting a misconfigured rule collection from event log evidence
- Troubleshooting a "file not found" error down to running commands on the wrong machine
- Mapping a technical control to the Essential Eight and documenting a known gap
