Commands Explained – Lab 16: Active Directory Basics

These are the commands from Lab 16, grouped by where I ran them. Each one shows the exact command and what every flag or argument does. Text in angle brackets, like <username>, is a placeholder: swap in your own value and leave out the brackets.


VirtualBox host (Command Prompt on my laptop)

```
VBoxManage natnetwork add --netname LabNet --network "<lab-subnet>/24" --enable --dhcp on
```
What it does: creates a VirtualBox NAT Network, a private network the VMs share. The VMs can reach each other and the internet, but not my home network.
- VBoxManage: VirtualBox's command-line tool
- natnetwork add: create a new NAT network
- --netname LabNet: the name of the network
- --network "<lab-subnet>/24": the address range. /24 means the first three numbers are fixed and the last one changes (up to 254 usable addresses).
- --enable: switch the network on
- --dhcp on: hand out IP addresses automatically to VMs that ask for one

```
VBoxManage natnetwork list
```
What it does: lists all NAT networks so you can confirm LabNet exists and is enabled.
- natnetwork list: show every NAT network and its settings

Tip: if VBoxManage isn't recognised, run the command from the VirtualBox install folder (usually C:\Program Files\Oracle\VirtualBox) or add that folder to PATH.


Any Windows machine (Command Prompt or PowerShell)

```
ipconfig /all
```
What it does: shows full network settings for every adapter.
- ipconfig: shows IP configuration
- /all: include everything: DNS servers, MAC address, DHCP on or off, and more. Used to confirm DC01's static IP and that its DNS pointed at itself.

```
nslookup silkylab.local
```
What it does: asks the DNS server to resolve a name. If the domain name resolves to the DC, AD's DNS is working.
- nslookup: DNS lookup tool
- silkylab.local: the name to look up

```
nslookup -type=SRV _ldap._tcp.dc._msdcs.silkylab.local
```
What it does: looks up the SRV record clients use to find a domain controller. If this fails, PCs can't join or log in to the domain.
- -type=SRV: look for a service record instead of a normal address record
- _ldap._tcp.dc._msdcs.silkylab.local: the standard name for "the domain controllers of this domain"

```
ping DC01
```
What it does: checks that one machine can reach another over the network. In the clock-skew fault, a working ping told me the network was fine, so the problem was somewhere else.
- DC01: the name (or IP) of the machine to test

```
ncpa.cpl
```
What it does: opens Network Connections directly. I used it to set DC01's static IP and to point Win11's DNS at DC01.

```
dsa.msc
```
What it does: opens Active Directory Users and Computers (ADUC), the main tool for managing users, groups, computers and OUs. On a DC it's built in. On a client it needs RSAT.

```
gpmc.msc
```
What it does: opens Group Policy Management, where you create GPOs and link them to OUs.


Domain controller (PowerShell as Administrator)

```
Get-ADDomain
```
What it does: shows the domain's details: DNS name, NetBIOS name, forest, domain mode and which DC holds the key roles. If this works, the promotion worked.
- No arguments needed: by default it shows the domain the computer belongs to

```
Get-ADGroupMember -Identity Lab-Helpdesk
```
What it does: lists the members of a group.
- -Identity Lab-Helpdesk: the group to check

```
Get-ADComputer -Identity <client-computer-name>
```
What it does: looks up a computer account in AD. If it returns details, the domain join created the account.
- -Identity: the computer name to look up

```
Get-ADUser <username> | Select DistinguishedName
```
What it does: shows the user's full path in AD, which tells you which OU they're in. Useful when a GPO applies to one person but not another.
- Get-ADUser <username>: look up the user account
- | (pipe): pass the result to the next command
- Select DistinguishedName: show only the full AD path, e.g. CN=tuser,OU=...,DC=silkylab,DC=local

```
Get-TimeZone
```
What it does: shows the time zone Windows is set to.

```
Set-TimeZone -Id "AUS Eastern Standard Time"
```
What it does: changes the time zone.
- -Id "AUS Eastern Standard Time": Windows' internal name for the Canberra, Melbourne, Sydney zone. Get-TimeZone -ListAvailable shows all the names.

```
w32tm /config /manualpeerlist:"time.windows.com,0x8" /syncfromflags:manual /reliable:yes /update
```
What it does: makes the DC the domain's trusted clock and gives it an internet time server to sync with.
- w32tm: the Windows Time service tool
- /config: change the time settings
- /manualpeerlist:"time.windows.com,0x8": the time server to use. 0x8 means "ask it as a normal client".
- /syncfromflags:manual: use the manual list above. The first DC is the top of the domain and has no one above it to ask.
- /reliable:yes: announce this DC as a reliable time source for the domain
- /update: tell the time service to load the new settings now

```
Restart-Service w32time
```
What it does: restarts the Windows Time service so the new settings definitely take effect.
- w32time: the service name of Windows Time

```
w32tm /resync /force
```
What it does: syncs the clock right now.
- /resync: sync immediately instead of waiting for the next scheduled sync
- /force: sync even if the time difference is large

```
w32tm /query /status
```
What it does: shows where the clock gets its time from and when it last synced.
- /query: ask for information
- /status: show source, last successful sync time and accuracy


Windows 11 client

```
whoami /groups
```
What it does: shows the logged-in account and every group it belongs to. I used it to confirm SILKYLAB\tuser was in Lab-Helpdesk.
- whoami: who am I logged in as
- /groups: list group memberships too

```
gpupdate /force
```
What it does: refreshes Group Policy now instead of waiting for the background refresh (about every 90 minutes).
- gpupdate: Group Policy update
- /force: reapply all policies, not just the ones that changed

```
gpresult /r /scope:user
```
What it does: shows which Group Policies applied. Run it in the affected user's own session.
- gpresult: Group Policy results
- /r: short summary report
- /scope:user: show user policies only (/scope:computer shows the computer half)

```
gpresult /h C:\Temp\gpreport.html
```
What it does: saves a full, detailed policy report as a web page, handy to attach to a ticket. (Optional, not run in this lab.)
- /h: write an HTML report to the path given

```
w32tm /query /source
```
What it does: shows which machine this PC gets its time from. On a domain PC it should be the DC (DC01.silkylab.local).
- /source: show only the time source

```
w32tm /resync
```
What it does: syncs the clock with its source (the DC) right now.


PowerShell equivalents of the GUI steps (for reference – I used the GUI for these in Lab 16)

```
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```
- Installs the AD DS role. -IncludeManagementTools also installs ADUC and the AD PowerShell module.

```
Install-ADDSForest -DomainName "silkylab.local" -DomainNetbiosName "SILKYLAB" -InstallDns
```
- Promotes the server to the first DC of a new forest
- -DomainName: the domain's DNS name
- -DomainNetbiosName: the short (pre-Windows 2000) name
- -InstallDns: install DNS on the DC as well. It prompts for the DSRM password.

```
New-ADOrganizationalUnit -Name "<OU-name>"
```
- Creates an OU. -Name is the OU's name.

```
New-ADUser -Name "tuser" -AccountPassword (Read-Host -AsSecureString "Password") -Enabled $true -ChangePasswordAtLogon $true -Path "OU=<OU-name>,DC=silkylab,DC=local"
```
- Creates a user
- -AccountPassword (Read-Host -AsSecureString ...): prompts for the password without showing it on screen
- -Enabled $true: the account is usable straight away
- -ChangePasswordAtLogon $true: forces a password change at first logon
- -Path: the OU to create the user in

```
New-ADGroup -Name "Lab-Helpdesk" -GroupScope Global -GroupCategory Security
```
- Creates a group
- -GroupScope Global: the usual scope for grouping users
- -GroupCategory Security: a group that can be given permissions (a Distribution group is for email only)

```
Add-ADGroupMember -Identity Lab-Helpdesk -Members tuser
```
- Adds tuser to the group

```
Add-Computer -DomainName silkylab.local -Restart
```
- Joins the PC to the domain (run on the client as admin). It prompts for domain credentials.
- -Restart: reboot straight away to finish the join
