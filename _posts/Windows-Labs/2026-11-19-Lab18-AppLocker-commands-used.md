Lab 18: AppLocker (Essential Eight application control)
=======================================================

Note: DESKTOP-XXXXXXX stands for the workstation's real computer name. Swap in your own (find it with the first command below).


Part A: Put the workstation in its own OU
-----------------------------------------

Runs on: DC01, Terminal (Admin)

```
Get-ADComputer -Filter * | Select Name
```
What it does: lists every computer account in Active Directory and shows only their names.
- Get-ADComputer: the PowerShell cmdlet that reads computer accounts from Active Directory
- -Filter: tells it which computers to return
- `*`: a wildcard meaning "all of them"
- `|`: the pipe, which hands the results to the next command
- Select: short for Select-Object; keeps only the columns you name
- Name: the column to keep, the computer's name

```
New-ADOrganizationalUnit -Name "Lab-Workstations" -Path "DC=silkylab,DC=local"
```
What it does: creates a new OU called Lab-Workstations at the top level of the domain.
- New-ADOrganizationalUnit: the cmdlet that creates an Organizational Unit (OU) in Active Directory
- -Name: the name of the new OU
- "Lab-Workstations": the name itself, in quotes so it's read as one piece of text
- -Path: where in AD to create it
- "DC=silkylab,DC=local": the domain root, written as a distinguished name. DC means Domain Component, so this is silkylab.local split into its two parts, with the comma separating them

```
$pc = Get-ADComputer "DESKTOP-XXXXXXX"
```
What it does: finds the workstation's computer account and saves it in a variable so the next command stays short.
- $pc: a variable named pc. The `$` marks it as a variable (a labelled box that holds a result)
- `=`: puts the result on the right into the variable on the left
- Get-ADComputer: reads a computer account from Active Directory
- "DESKTOP-XXXXXXX": the computer's name, in quotes

```
Move-ADObject $pc -TargetPath "OU=Lab-Workstations,DC=silkylab,DC=local"
```
What it does: moves the workstation out of the default Computers container and into the Lab-Workstations OU, so a GPO can be linked to it.
- Move-ADObject: the cmdlet that moves any object (user, computer, group) to a different place in AD
- $pc: the computer account saved by the previous command
- -TargetPath: where to move it to
- "OU=Lab-Workstations,DC=silkylab,DC=local": the destination as a distinguished name. OU= is the Organizational Unit, and the DC= parts are the domain it sits in. Commas separate each part

```
(Get-ADComputer "DESKTOP-XXXXXXX").DistinguishedName
```
What it does: shows the computer's full AD location so you can confirm the move worked.
- `( )`: run the command inside first, then use its result
- Get-ADComputer: reads the computer account
- "DESKTOP-XXXXXXX": the computer's name
- `.`: reaches into the result to pick out one property
- DistinguishedName: the property holding the object's full path in AD. It should now include OU=Lab-Workstations


Part B: Open Group Policy Management
------------------------------------

Runs on: DC01, Start, Run (Win+R)

```
gpmc.msc
```
What it does: opens the Group Policy Management console, where GPOs are created, edited and linked to OUs.
- gpmc: Group Policy Management Console
- .msc: Microsoft Management Console file, the file type for Windows admin consoles


Part C: Apply the policy and read audit events
----------------------------------------------

Runs on: Win11-Ent-Lab, signed in as SILKYLAB\Administrator, Terminal (Admin)

```
gpupdate /force
```
What it does: makes the PC download and reapply every Group Policy setting now, instead of waiting for the normal refresh (about every 90 minutes).
- gpupdate: the command-line tool that refreshes Group Policy
- /force: reapply all policies, not just the ones that changed

```
gpresult /r /scope:computer
```
What it does: shows a summary of which GPOs applied to this computer, so you can confirm Lab-AppLocker is in the list.
- gpresult: the tool that reports which Group Policy settings applied
- /r: give a short summary report (not the full verbose one)
- /scope:computer: show computer policy only, not user policy. The colon joins the switch to its value

```
Get-Service AppIDSvc | Select Status, StartType
```
What it does: checks the Application Identity service, which AppLocker needs before any rule will work.
- Get-Service: the cmdlet that reads Windows services
- AppIDSvc: the short name of the Application Identity service
- `|`: hands the result to the next command
- Select: keeps only the columns you name
- Status: whether the service is running or stopped right now
- `,`: separates the two column names
- StartType: whether it starts Automatically, Manually or is Disabled. It should say Automatic

```
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-AppLocker/EXE and DLL';Id=8003} | Format-List TimeCreated,Message
```
What it does: finds audit-mode events showing a program was allowed but would have been blocked if AppLocker were enforced.
- Get-WinEvent: the cmdlet that reads Windows event logs (the command-line version of Event Viewer)
- -FilterHashtable: filters using a set of conditions inside the log itself, so it's fast
- `@{ }`: a hashtable, a list of name = value conditions. `@` marks the start, the curly braces hold the conditions
- LogName=: which log to search
- 'Microsoft-Windows-AppLocker/EXE and DLL': the AppLocker log for programs and DLLs. Single quotes because the name has spaces. In Event Viewer it's under Applications and Services Logs, Microsoft, Windows, AppLocker
- `;`: separates the two conditions
- Id=8003: the event ID for "allowed, but would have been blocked" (programs)
- `|`: hands the events to the next command
- Format-List: shows each event as a vertical list so the message isn't cut off
- TimeCreated: when the event happened
- `,`: separates the two property names
- Message: the full event text, including the file path

```
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-AppLocker/MSI and Script';Id=8006} | Format-List TimeCreated,Message
```
What it does: the same check for scripts and installers, finding scripts that ran but would have been blocked if enforced.
- Same parts as the command above, except:
- 'Microsoft-Windows-AppLocker/MSI and Script': the AppLocker log for installers (.msi) and scripts (.bat, .cmd, .ps1, .vbs, .js)
- Id=8006: "allowed, but would have been blocked" (scripts and installers)


Part D: Enforce mode and checking a verdict
-------------------------------------------

Runs on: Win11-Ent-Lab, signed in as SILKYLAB\Administrator, Terminal (Admin). Not DC01: the Lab-AppLocker GPO only applies to the Lab-Workstations OU, so DC01 has no AppLocker rules.

```
Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-AppLocker/EXE and DLL';Id=8004} | Format-List TimeCreated,Message
```
What it does: finds enforce-mode events showing a program was actually blocked.
- Same parts as the 8003 command, except:
- Id=8004: "blocked" (programs). Use Id=8007 with the MSI and Script log for blocked scripts

```
Get-ChildItem C:\Users -Recurse -Filter test-app.exe -ErrorAction SilentlyContinue | Select-Object FullName
```
What it does: searches every user profile for a file by name and shows its full path, so you don't have to guess folder names like tuser or tuser.SILKYLAB.
- Get-ChildItem: the cmdlet that lists files and folders (like dir)
- C:\Users: the folder to start searching in
- -Recurse: also search every folder inside it, all the way down
- -Filter: only return items matching this name
- test-app.exe: the file name to look for
- -ErrorAction: what to do when an error happens
- SilentlyContinue: hide the error and keep going (skips folders you can't open)
- `|`: hands the results on
- Select-Object: keeps only the property you name
- FullName: the file's complete path

```
Copy-Item C:\Windows\System32\cmd.exe C:\Users\Public\test-app.exe
```
What it does: copies Command Prompt into the Public folder under a new name, to stand in for an unapproved downloaded program.
- Copy-Item: the cmdlet that copies a file (source first, then destination)
- C:\Windows\System32\cmd.exe: the source, the real Command Prompt program
- C:\Users\Public\test-app.exe: the destination, the shared Public folder with the new name. It's outside Windows and Program Files, so the default rules don't allow it

```
Get-AppLockerPolicy -Effective | Test-AppLockerPolicy -Path C:\Users\Public\test-app.exe -User SILKYLAB\tuser
```
What it does: asks the AppLocker rules in force on this PC whether tuser would be allowed to run the file, and which rule decides, without running it.
- Get-AppLockerPolicy: the cmdlet that reads AppLocker rules
- -Effective: the merged policy actually in force on this PC (GPO plus local)
- `|`: hands that policy to the next cmdlet
- Test-AppLockerPolicy: checks a file against the policy and gives the verdict
- -Path: which file to test
- C:\Users\Public\test-app.exe: the file being tested
- -User: which user to test as
- SILKYLAB\tuser: the domain, a backslash, then the username
- Result to expect: PolicyDecision DeniedByDefault with a blank MatchingRule (implicit deny: no rule allows it). Test C:\Windows\System32\notepad.exe the same way to see Allowed and the default rule that matched

```
Remove-Item C:\Users\Public\test-app.exe
```
What it does: deletes the test copy so no stray programs are left behind.
- Remove-Item: the cmdlet that deletes a file or folder
- C:\Users\Public\test-app.exe: the file to delete


AppLocker event ID quick map
----------------------------

| Result | Programs (EXE and DLL log) | Scripts and installers (MSI and Script log) |
|---|---|---|
| Allowed | 8002 | 8005 |
| Would have been blocked (audit mode) | 8003 | 8006 |
| Blocked (enforce mode) | 8004 | 8007 |
