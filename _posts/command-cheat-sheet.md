Command Cheat Sheet: CMD and PowerShell

My quick-lookup list of every command I've used in my Windows labs, so I don't have to remember the full syntax. Each entry gives the exact command, what it does in one line, and what each flag does. For the longer explanations and on-the-job notes, see my Commands Explained page.

Shell key: CMD = Command Prompt. PS = PowerShell. CMD/PS = works in both.
Admin = needs an elevated window (right-click Start > Terminal (Admin)).
Text in angle brackets, like <username>, is a placeholder: swap in your own value and leave out the brackets.

Last updated: Lab 17 (18 Nov 2026)

---

QUICK LAUNCH (Win + R, type it, press Enter)

ncpa.cpl – Network Connections (adapter settings, static IP, DNS servers) – Labs 03, 16
services.msc – Services (start, stop, restart a service) – Labs 08, 09
eventvwr.msc – Event Viewer – Labs 05, 10, 11
perfmon /rel – Reliability Monitor (crash and error history by day) – Lab 10
cleanmgr – Disk Cleanup – Lab 12
regedit – Registry Editor (look, don't touch unless you know why) – Lab 02
winver – shows the exact Windows edition and version – Lab 14
resmon – Resource Monitor (disk and network use per process) – Lab 07
control – Control Panel – Labs 11, 16
sysdm.cpl – System Properties (rename the PC, join a domain) – Lab 16
dsa.msc – Active Directory Users and Computers – Lab 16
gpmc.msc – Group Policy Management – Lab 16

Keyboard shortcuts
Ctrl + Shift + Esc – Task Manager – Labs 07, 09
Ctrl + Alt + Del – security screen; often still works when the desktop is frozen – Lab 07
Win + I – Settings – Lab 16
Win + Ctrl + Q – Quick Assist – Lab 15

---

NETWORKING AND DNS

ipconfig   (CMD/PS)   Labs 03, 06, 16
```
ipconfig /all
```
Full IP settings for every adapter: IP, gateway, DNS servers, DHCP on/off, MAC.
- /all: show everything, not just the short summary

```
ipconfig /flushdns
```
Clears the cached DNS answers, so the next lookup asks the DNS server fresh.
- /flushdns: empty the DNS cache

ping   (CMD/PS)   Labs 03, 06, 16
```
ping 127.0.0.1
ping <gateway-IP>
ping 8.8.8.8
ping google.com
```
Tests whether you can reach something. Work bottom up: yourself (127.0.0.1), the gateway, the internet by IP, then by name. If the IP works but the name fails, it's DNS.

nslookup   (CMD/PS)   Labs 03, 06, 16
```
nslookup google.com
```
Asks your DNS server to look up a name.

```
nslookup google.com 8.8.8.8
```
Asks a specific DNS server instead of your default one. Handy to prove whether your DNS server is the problem.
- 8.8.8.8: the DNS server to ask

```
nslookup -type=SRV _ldap._tcp.dc._msdcs.<domain>
```
Finds the domain controllers for a domain. If this fails, PCs can't join or log in.
- -type=SRV: look up a service record instead of an address record

---

PROCESSES, SERVICES AND PERFORMANCE

tasklist   (CMD/PS)   Lab 09
```
tasklist /svc /fi "imagename eq svchost.exe"
```
Lists every svchost.exe with the services running inside it.
- /svc: show the services hosted in each process
- /fi "imagename eq svchost.exe": filter, only processes named svchost.exe (eq = equals)

```
tasklist /svc /fi "PID eq <PID>"
```
Shows which service is inside one process ID, e.g. the busy one from Task Manager.
- /fi "PID eq <PID>": filter by process ID

Get-Process   (PS)   Lab 09
```
Get-Process svchost | Select Id, Path
```
Shows each svchost's ID and where it runs from. Genuine ones are in C:\Windows\System32.
- svchost: the process name (no .exe)
- | Select Id, Path: show only the ID and file path

```
(Get-Process services).Id
```
Gives the process ID of services.exe, the parent of every genuine svchost.

```
Get-Process -Id <PID>
```
Looks up one process by its ID, e.g. to find out what a parent process is.
- -Id: the process ID to look up

Get-CimInstance   (PS)   Lab 09
```
Get-CimInstance Win32_Process -Filter "name='svchost.exe'" | Select ProcessId, ParentProcessId, ExecutablePath
```
Shows each svchost with its parent ID and full path. Get-Process can't show the parent, which is why this one is needed.
- Win32_Process: the class of Windows information to read (running processes)
- -Filter "name='svchost.exe'": only processes with this name
- | Select ...: show only these three columns

```
Get-CimInstance Win32_Process -Filter "name='svchost.exe'" | Where-Object { $_.ExecutablePath -notlike "C:\Windows\System32\*" } | Format-Table ProcessId, ParentProcessId, ExecutablePath -AutoSize
```
Hunts for fake svchosts: shows only ones not running from System32. No results = good.
- Where-Object { ... }: keep only the items that match the test in the braces
- $_.ExecutablePath: the path of the current item
- -notlike "C:\Windows\System32\*": does not start with that folder (the star is a wildcard)
- Format-Table ... -AutoSize: show as a neat table with columns sized to fit

Read it as a sentence: "Get every svchost, keep the ones not in System32, show me their ID, parent and path."

Restart-Service   (PS, Admin)   Lab 16
```
Restart-Service <service-name>
```
Stops and starts a service, e.g. Restart-Service w32time or Restart-Service Spooler (Print Spooler, Lab 08). The GUI version is services.msc > right-click > Restart.

CPU stress test (lab use only)   (PS)   Lab 07
```
while ($true) {}
```
An endless loop that keeps one CPU core busy, to practise finding a CPU hog. Stop it with Ctrl + C or by ending the process in Task Manager.
- while ($true): repeat for as long as true is true, which is forever
- {}: the work to repeat (nothing), so it just spins

---

USERS, GROUPS AND PASSWORDS (local accounts)

net user   (CMD/PS, Admin)   Labs 02, 04, 13
```
net user
```
Lists all local accounts.

```
net user <username>
```
Shows one account's details: active, locked, last logon, password dates, groups.

```
net user <username> <password> /add
```
Creates a local account.
- /add: create it

```
net user <username> *
```
Resets the password and prompts you to type it, so it isn't shown on screen.
- *: prompt for the password instead of typing it in the command

```
net user <username> /logonpasswordchg:yes
```
Forces a password change at next logon.
- /logonpasswordchg:yes: must change the password at next sign-in

```
net user <username> /delete
```
Deletes the account.

net accounts   (CMD/PS, Admin)   Lab 04
```
net accounts
```
Shows the password and lockout policy.

```
net accounts /lockoutthreshold:3
net accounts /lockoutwindow:5
net accounts /lockoutduration:10
```
Sets the lockout policy.
- /lockoutthreshold:3: lock the account after 3 wrong passwords
- /lockoutwindow:5: count wrong attempts within 5 minutes
- /lockoutduration:10: stay locked for 10 minutes

net localgroup   (CMD/PS, Admin)   Labs 04, 13
```
net localgroup <group>
```
Lists the members of a local group, e.g. Administrators.

```
net localgroup <group> /add
```
Creates a new local group (no username = create the group).

```
net localgroup <group> <username> /add
net localgroup <group> <username> /delete
```
Adds a user to, or removes a user from, a group.

whoami   (CMD/PS)   Labs 13, 16
```
whoami
```
Shows who you're logged in as (domain\user or computer\user).

```
whoami /groups
```
Lists every group in your current logon. A newly added group won't show until you log off and on.
- /groups: include group memberships

```
whoami /groups | findstr Share
```
Shows only the lines containing "Share".
- | findstr Share: filter the output for a word (CMD's version of a search)

runas   (CMD)   Lab 13
```
runas /user:<username> cmd
```
Opens a Command Prompt as a different user, to test what they can and can't do.
- /user:<username>: the account to run as
- cmd: the program to open

---

FILES, SHARES AND PERMISSIONS

Basic file commands   (CMD)   Lab 13
```
cd C:\LabShare
mkdir C:\LabShare
dir /b C:\LabShare
echo This is a test file > test.txt
echo More text >> test.txt
type test.txt
exit
```
- cd: change folder
- mkdir: make a folder
- dir /b: list files, bare format (names only)
- echo text > file: write text to a file (replaces what's there)
- echo text >> file: add text to the end of a file
- type: show a file's contents
- exit: close the window

Copy-Item   (PS)   Lab 09
```
Copy-Item <source> <destination>
```
Copies a file, e.g. Copy-Item C:\Windows\System32\cmd.exe "$env:USERPROFILE\Desktop\svchost.exe"
- $env:USERPROFILE: your user folder (C:\Users\yourname) without typing your name

net share   (CMD, Admin)   Lab 13
```
net share LabShare=C:\LabShare /grant:Everyone,FULL
```
Shares a folder on the network.
- LabShare=C:\LabShare: share name = local folder
- /grant:Everyone,FULL: share permission. Real security is set with NTFS permissions (icacls).

net use   (CMD)   Lab 13
```
net use S: \\<computer>\<share> /persistent:yes
```
Maps a network drive.
- S:: the drive letter
- \\<computer>\<share>: the UNC path to the share (%COMPUTERNAME% = this PC)
- /persistent:yes: reconnect it at every logon

```
net use
```
Lists mapped drives and their status. System error 67 = the share name or path is wrong (often a typo).

icacls   (CMD, Admin)   Lab 13
```
icacls C:\LabShare
```
Shows who has which NTFS permissions on a folder.

```
icacls C:\LabShare /grant <group>:(OI)(CI)M
```
Gives a group permission on the folder and everything inside it.
- (OI): object inherit, applies to files inside
- (CI): container inherit, applies to subfolders
- F = Full control, M = Modify, RX = Read and execute

```
icacls C:\LabShare /inheritance:r
```
Removes permissions inherited from the parent folder, so only the ones you set apply. Grant your own permissions first, or you can lock yourself out.
- /inheritance:r: remove inherited permissions

---

DISK AND STORAGE

fsutil   (CMD, Admin)   Lab 12
```
fsutil file createnew C:\Users\<username>\Downloads\old_backup.zip 5368709120
```
Creates a dummy file of an exact size (5 GB here) to practise a full drive.
- file createnew: make a new empty file
- 5368709120: size in bytes (5 x 1024 x 1024 x 1024)

---

BITLOCKER AND TPM

Get-Tpm   (PS, Admin)   Lab 14
```
Get-Tpm
```
Checks the TPM chip is present and ready. Look for TpmPresent and TpmReady: True.

manage-bde   (CMD/PS, Admin)   Labs 05, 14
```
manage-bde -status C:
```
Shows encryption status, percentage encrypted, and whether protection is on.

```
manage-bde -protectors -get C:
```
Lists the drive's protectors (TPM, recovery password) and shows the 48-digit recovery key. Check this before relying on a key.
- -protectors -get: show the protectors

```
manage-bde -protectors -add C: -RecoveryPassword
manage-bde -protectors -add C: -TPM
```
Adds a recovery password, or adds the TPM back.
- -RecoveryPassword: generate a 48-digit recovery key
- -TPM: use the TPM chip to unlock the drive automatically at boot

```
manage-bde -protectors -enable C:
```
Turns protection on (resumes BitLocker).

```
manage-bde -forcerecovery C:
```
Forces the recovery screen at next boot (lab use: practise entering the key). It removes the TPM protector, so add it back afterwards.

---

ACTIVE DIRECTORY (PowerShell on the DC, or a PC with RSAT)

```
Get-ADDomain
```
Shows the domain's name, NetBIOS name, forest and key DC roles. (Lab 16)

```
Get-ADComputer -Filter *
```
Lists every computer account. Confirms a PC joined the domain. (Lab 16)
- -Filter *: return everything (* = all)

```
Get-ADGroupMember -Identity "<group>"
```
Lists who is in a group. (Lab 16)
- -Identity: which group

```
Get-ADUser <username> | Select DistinguishedName
```
Shows which OU a user lives in. First check when a GPO applies to one person but not another. (Lab 16)
- | Select DistinguishedName: show only the full AD path

---

GROUP POLICY

```
gpupdate /force
```
Applies Group Policy now instead of waiting about 90 minutes. (CMD/PS, Lab 16)
- /force: reapply every policy, not just changed ones

```
gpresult /r /scope:user
```
Shows which user policies applied. Run it in the affected user's own session. (CMD/PS, Lab 16)
- /r: summary report
- /scope:user: user policies only (/scope:computer for the computer half, needs Admin)

---

TIME (Kerberos fails if clocks are more than 5 minutes apart)

```
Get-TimeZone
Set-TimeZone -Id "AUS Eastern Standard Time"
```
Shows, then sets, the time zone (Canberra, Melbourne, Sydney). (PS, Lab 16)
- -Id: Windows' internal name for the zone. Get-TimeZone -ListAvailable lists them all.

```
w32tm /query /status
w32tm /query /source
```
Shows where the clock gets its time, and when it last synced. On a domain PC the source should be the DC. (CMD/PS, Lab 16)

```
w32tm /resync /force
```
Syncs the clock now. (Admin, Lab 16)
- /force: sync even if the time gap is large

```
w32tm /config /manualpeerlist:"time.windows.com,0x8" /syncfromflags:manual /reliable:yes /update
```
Makes the first DC sync with an internet time server and act as the domain's trusted clock. Run Restart-Service w32time afterwards. (Admin, on the DC only, Lab 16)
- /manualpeerlist: the time server to use (0x8 = ask as a normal client)
- /syncfromflags:manual: use that list
- /reliable:yes: advertise this DC as a trusted time source
- /update: load the new settings now

---

AUDIT POLICY AND SECURITY LOG (on the DC, Admin)

```
auditpol /get /category:*
```
Shows the audit policy the machine is actually using, every subcategory. The source of truth, not the GPO. (CMD/PS, Lab 17)
- /get: display current settings
- /category:*: all categories

```
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4768} -MaxEvents 5
```
Newest events of one ID from the Security log. Swap the Id for any event. (PS, Lab 17)
- -FilterHashtable: filter at the source (fast)
- LogName / Id: which log, which event
- -MaxEvents: how many, newest first

```
(Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 1).Message
```
Full text of the newest event, with every field. (PS, Lab 17)
- .Message: full text instead of the summary

```
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4672} -MaxEvents 5 | Format-List TimeCreated, Message
```
Newest privileged (admin) logons. (PS, Lab 17)
- Format-List: show chosen properties one per line

```
$events = Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 20
$events | Select-Object TimeCreated, @{n='Account';e={$_.Properties[5].Value}}, @{n='LogonType';e={$_.Properties[8].Value}}, @{n='SourceIP';e={$_.Properties[18].Value}} | Format-Table -AutoSize
```
Logon events as a table: when, who, how, from where. (PS, Lab 17)
- $events =: store results in a variable to reuse
- @{n='name';e={...}}: calculated column (n = name, e = expression)
- 4624 data positions: [5] account, [8] logon type, [10] Kerberos or NTLM, [18] source IP

Key event IDs: 4624 logon OK, 4625 logon failed, 4672 admin logon, 4720 user created, 4724 password reset, 4728/4732 added to group, 4740 locked out, 4768 Kerberos TGT, 4771 Kerberos pre-auth failed, 4776 NTLM check
Logon types: 2 keyboard, 3 network, 5 service, 10 RDP

---

VIRTUALBOX (host laptop, CMD)

```
VBoxManage natnetwork add --netname LabNet --network "<lab-subnet>/24" --enable --dhcp on
VBoxManage natnetwork list
```
Creates a NAT Network for lab VMs, then lists NAT networks to check it worked. (Lab 16)
- --netname: the network's name
- --network: the address range
- --enable: switch it on
- --dhcp on: hand out addresses automatically

---

OUTLOOK (classic Outlook only)

```
outlook.exe /safe
```
Starts classic Outlook without add-ins. If it works in safe mode, an add-in is the problem. Not supported in new Outlook. (Lab 11)


Check Disc Space 

Get-PSDrive C

What it does: shows how much space is used and free on the C: drive.

Get-PSDrive – the PowerShell cmdlet that lists drives and how much space each one has
C – the drive letter, so it shows only the C: drive instead of every drive
Output to read – Used (GB) and Free (GB) columns

When I use it: checking there's enough free space before building a new VM, and confirming space came back after a clean-up.


PowerShell tips for not memorising
- Tab completion: type Get-Pro and press Tab. It works for parameters too.
- Get-Help <command> -Examples shows example commands.
- | Select <columns> keeps output short. | Format-Table -AutoSize makes it readable.

- More commands will be added as I go though them. 
