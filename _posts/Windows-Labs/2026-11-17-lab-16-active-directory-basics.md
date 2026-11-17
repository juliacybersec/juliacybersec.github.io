---
title: "Lab 16: Active Directory Basics – Building a Domain Controller and Joining a Windows 11 Client"
date: 2026-11-17
---

Lab 16: Active Directory Basics – Building a Domain Controller and Joining a Windows 11 Client

Lab dates: 11–17 November 2026
Environment: VirtualBox on my laptop – Windows Server 2025 (evaluation) and Windows 11 Enterprise (evaluation), on an isolated NAT network


What I Built

I built a small Active Directory domain from scratch:

- A Windows Server 2025 VM (DC01), promoted to a domain controller for a new forest, silkylab.local
- An organisational unit (OU), a test user (tuser) and a security group (Lab-Helpdesk)
- A Windows 11 Enterprise VM joined to the domain, with tuser signing in as a domain user
- BONUS: a Group Policy Object (Lab-Block-ControlPanel) that blocks Control Panel and Settings for users in the lab OU
- A proper time sync setup, after a real clock-skew problem broke Group Policy part way through the lab

[Screenshot: lab16-01-virtualbox-dc01-and-win11-on-labnet.png]


Why It Matters

Most help desk tickets I handled earlier in my career at ANZ were about accounts, logins and profiles. Active Directory sits behind nearly all of them: password resets, locked accounts, "I can't log in on this PC", missing drive mappings and policies that apply to one person but not another. On the help desk I worked with the results of AD. In this lab I built the directory myself, so I understand what is happening behind those tickets.

At a strategic level, this is also the foundation of most Windows security and compliance work. If I am assessing a mid-size organisation's controls, I need to know how identity, DNS, time and Group Policy fit together, because many security baselines are enforced through exactly these tools.


Lab Setup

- VirtualBox NAT Network called LabNet, so both VMs can talk to each other and reach the internet, but stay isolated from my home network
- DC01: Windows Server 2025, 4 GB RAM, 2 CPUs, EFI, 50 GB dynamic disk
- Win11-Ent-Lab: Windows 11 Enterprise evaluation (the same VM I used for my BitLocker and Quick Assist labs)

Note: I have left IP addresses and passwords out of this write-up on purpose.


Steps (summary)

Full explanations of every command, flag and argument are on my Commands Explained page.

Part A – Build and prepare DC01

1. Created the LabNet NAT network from the command line (the GUI option gave me trouble):

```
VBoxManage natnetwork add --netname LabNet --network "<lab-subnet>/24" --enable --dhcp on
```

   - natnetwork add: create a new NAT network
   - --netname LabNet: the network's name
   - --network: the address range the network uses
   - --enable: turn the network on
   - --dhcp on: hand out addresses automatically to VMs that need them

2. Connected both VMs to LabNet (VM Settings > Network > Attached to: NAT Network > LabNet).
3. Installed Windows Server 2025 on DC01.
4. Renamed the server to DC01.
5. Set a static IP in ncpa.cpl, with DNS pointing to DC01's own address. A domain controller must never change address, because every client finds it through DNS.
6. Confirmed the settings:

```
ipconfig /all
```

   - /all: show full details, including DNS servers, not just the IP address

7. Installed the Active Directory Domain Services (AD DS) role using the Add Roles and Features wizard.

[Screenshot: lab16-02-dc01-static-ip-dns-self.png]
[Screenshot: lab16-03-add-roles-ad-ds.png]

Part B – Promote DC01 to a domain controller and verify

1. Ran "Promote this server to a domain controller" from Server Manager.
2. Chose "Add a new forest", with the root domain name silkylab.local (NetBIOS name SILKYLAB).
3. Set the DSRM (Directory Services Restore Mode) password and let the server reboot.
4. Logged in as SILKYLAB\Administrator and verified the domain:

```
Get-ADDomain
```

   - Shows the domain's details: name, NetBIOS name, forest, and which DC holds the key roles

```
nslookup silkylab.local
```

   - Asks DNS to resolve the domain name. A reply from DC01 means AD's DNS is working.

[Screenshot: lab16-04-promote-new-forest.png]
[Screenshot: lab16-05-get-addomain.png]
[Screenshot: lab16-06-nslookup-domain.png]

Part C – Create an OU, a user and a group

1. In Active Directory Users and Computers (dsa.msc), created an OU for lab users.
2. Created a test user, tuser, with "User must change password at next logon" ticked, the same way a help desk sets up a new starter.
3. Created a security group, Lab-Helpdesk, and added tuser to it.
4. Confirmed the membership:

```
Get-ADGroupMember -Identity Lab-Helpdesk
```

   - -Identity Lab-Helpdesk: the group to list members of

[Screenshot: lab16-07-aduc-ou-user-group.png]
[Screenshot: lab16-08-get-adgroupmember.png]

Part D – Join Windows 11 to the domain and log in as a domain user

1. On Win11-Ent-Lab, changed the DNS server (ncpa.cpl) to DC01's IP. Clients must use the DC for DNS, otherwise they can't find the domain's SRV records.
2. Joined the PC to silkylab.local (System Properties > Change > Domain) and restarted.
3. On DC01, confirmed the computer account had been created:

```
Get-ADComputer -Identity <client-computer-name>
```

   - -Identity: the computer name to look up. If it returns details, the join worked.

4. Logged in to Win11-Ent-Lab as SILKYLAB\tuser and was forced to change the password at first logon.
5. Checked the user's identity and groups:

```
whoami /groups
```

   - whoami: shows who is logged in
   - /groups: also lists every group the account belongs to. Lab-Helpdesk appeared.

[Screenshot: lab16-09-win11-dns-to-dc01.png]
[Screenshot: lab16-10-domain-join-welcome.png]
[Screenshot: lab16-11-get-adcomputer.png]
[Screenshot: lab16-12-tuser-forced-password-change.png]
[Screenshot: lab16-13-whoami-groups.png]

Part E – BONUS: Group Policy to block Control Panel and Settings

1. On DC01, opened Group Policy Management (gpmc.msc).
2. Created a GPO called Lab-Block-ControlPanel and linked it to the lab users OU.
3. Edited the GPO: User Configuration > Policies > Administrative Templates > Control Panel > Prohibit access to Control Panel and PC settings > Enabled.
4. On Win11-Ent-Lab, as tuser, refreshed policy:

```
gpupdate /force
```

   - gpupdate: refreshes Group Policy now instead of waiting for the background refresh
   - /force: reapplies every policy, not just the ones that changed

5. Tested: Win + R > control gave "This operation has been cancelled due to restrictions in effect on this computer", and Win + I (Settings) would not open.
6. Proved the policy applied:

```
gpresult /r /scope:user
```

   - /r: short summary report
   - /scope:user: user policies only. Lab-Block-ControlPanel was listed under Applied Group Policy Objects.

[Screenshot: lab16-14-gpmc-gpo-linked.png]
[Screenshot: lab16-15-gpo-prohibit-control-panel.png]
[Screenshot: lab16-16-control-panel-blocked.png]
[Screenshot: lab16-17-gpresult-applied.png]


Troubleshooting: When Group Policy Failed Because of the Clock

This was the most useful part of the lab, because it was a real fault I had to work out myself.

Symptom: after both VMs had sat idle over lunch, gpupdate /force failed on Win11-Ent-Lab.

What I checked:
- DC01 was running
- ping to DC01 worked, so the network and name resolution were fine
- then I compared the clocks: the two VMs were 7 minutes apart, and both were hours behind the real time

Cause: Kerberos, the authentication protocol AD uses, rejects any request whose timestamp differs from the DC's clock by more than 5 minutes (the default maximum clock skew). Even with correct credentials, the client could not authenticate to the DC to pull its policy.

Short-term fix: restarting both VMs brought the clocks close enough, and gpupdate /force succeeded.

Permanent fix:

On DC01 (Terminal as Administrator):

```
Set-TimeZone -Id "AUS Eastern Standard Time"
w32tm /config /manualpeerlist:"time.windows.com,0x8" /syncfromflags:manual /reliable:yes /update
Restart-Service w32time
w32tm /resync /force
w32tm /query /status
```

On Win11-Ent-Lab (as SILKYLAB\Administrator):

```
Set-TimeZone -Id "AUS Eastern Standard Time"
w32tm /query /source
w32tm /resync
```

The wrong time zone explained why both machines looked "hours behind". DC01 is now the domain's trusted time source and syncs with an internet time server, and the Windows 11 client syncs from DC01 (w32tm /query /source shows DC01.silkylab.local). Every flag is explained on my Commands Explained page.

[Screenshot: lab16-18-gpupdate-failed-time-skew.png]
[Screenshot: lab16-19-w32tm-query-status-dc01.png]
[Screenshot: lab16-20-w32tm-source-win11.png]


Key Learnings

What is DSRM?
Directory Services Restore Mode is a special boot mode for a domain controller in which the AD database is offline, so it can be repaired or restored. Because AD isn't running, normal domain accounts can't log in, which is why it has its own separate password, set during promotion. That password needs to be stored securely.

Why does a DC point its DNS at itself?
Clients find a domain controller by asking DNS for SRV records (for example _ldap._tcp.dc._msdcs.silkylab.local). Those records live in the DNS server that AD installs on the DC. The DC has to use itself for DNS so it can register and find its own records. Clients must point at the DC for the same reason: if a PC uses the home router for DNS, it can't find the domain and the join or login fails.

Why did the policy block tuser but not Administrator on the same PC?
The GPO is linked to the OU containing tuser, and it is a User Configuration setting, so it follows tuser to any domain PC. The Administrator account sits in the default Users container, which is not an OU and can't have GPOs linked to it, so the policy never reaches it. Computer Configuration settings work the other way: they stick to the machine, whoever logs in.

Why does a 5-minute clock difference break domain logons?
Kerberos includes an encrypted timestamp in each authentication request. The DC rejects anything more than 5 minutes from its own clock. This stops replay attacks: a captured logon message soon becomes stale, and an attacker can't re-timestamp it without the user's key.

Help desk scenario: one colleague is blocked from Settings and the other isn't, but both should be
1. Run gpresult /r /scope:user in each user's session and compare the applied GPOs.
2. Check which OU each account lives in, in ADUC or with Get-ADUser <username> | Select DistinguishedName. The most likely cause is an account in the wrong OU.
3. Move the account to the correct OU, then run gpupdate /force and log off and on.
4. Verify with gpresult again and document the ticket.
If both accounts are in the right OU, check the GPO's security filtering next.


Skills Demonstrated

- Building a Windows Server 2025 domain controller and a new AD forest
- DNS for Active Directory (static IP, DC as DNS server, SRV records, verifying with nslookup)
- Creating and managing OUs, users and groups in ADUC, and verifying with PowerShell
- Joining a Windows 11 client to a domain and verifying the computer account
- Creating, linking, testing and verifying a Group Policy Object (gpupdate, gpresult)
- Real-world troubleshooting: tracing a Group Policy failure to Kerberos clock skew, and fixing it permanently with time zone and NTP (w32tm) configuration
- VirtualBox networking from the command line (VBoxManage)


What's Next

- A short follow-up Group Policy lab
- Entra ID and Intune labs (cloud identity and device management)
- PowerShell for everyday help desk tasks
