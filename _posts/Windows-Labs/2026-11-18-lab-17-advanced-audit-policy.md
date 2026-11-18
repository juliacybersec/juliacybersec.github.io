---
title: "Lab 17: Advanced Audit Policy – Making a Domain Controller Detection-Ready"
date: 2026-11-18
---

Lab 17: Advanced Audit Policy – Making a Domain Controller Detection-Ready

Lab date: 18 November 2026
Environment: VirtualBox on my laptop – Windows Server 2025 domain controller (DC01) in my silkylab.local domain, on an isolated NAT network


What I Built

I took the domain controller I built in Lab 16 and made it detection-ready:

- Baselined DC01's effective audit policy with auditpol, to see what an out-of-the-box DC does and doesn't record
- Built a dedicated GPO, Lab-DC-Audit-Policy, linked to the Domain Controllers OU, that turns on advanced auditing for credential validation, Kerberos authentication, user and group changes, logons and privileged logons
- Enforced advanced audit policy over the legacy audit settings, applied the GPO and verified the result
- Queried and parsed the Security log with PowerShell (Get-WinEvent), and built a reusable table of logon events showing account, logon type and source address
- Worked through a triage scenario: an admin network logon at 2:14 am on a Sunday

[Screenshot: lab17-01-auditpol-baseline.png]


Why It Matters

A SIEM can only detect what the endpoints actually log. A domain controller is the most valuable logging source in a Windows environment, because every domain authentication passes through it. Yet out of the box, several of the events that matter most for spotting attacks are not fully recorded.

Getting audit policy right is the foundation that detection engineering, incident response and SIEM work all depend on. It is also something a security assessment checks early: "Are you logging the right events on your domain controllers, and is that enforced centrally?" This is the kind of control I would expect to review and document for a mid-size client, and to map against the Australian Signals Directorate's event logging guidance.


Lab Setup

- DC01: Windows Server 2025, domain controller for silkylab.local (built in Lab 16)
- VirtualBox Guest Additions installed on DC01 (VBoxWindowsAdditions-amd64.exe) and Shared Clipboard set to Bidirectional, so long PowerShell commands can be pasted in rather than retyped

Note: I have left IP addresses, hostnames beyond the lab names and passwords out of this write-up on purpose.


Steps (summary)

Full explanations of every command, flag and argument are on my Commands Explained page.

Part A – Baseline and build the audit GPO

1. Baselined the current effective audit policy on DC01 (Terminal as Administrator):

```
auditpol /get /category:*
```

   - auditpol: the built-in tool for viewing and setting Windows audit policy
   - /get: display the current settings
   - /category:*: every audit category (* means all)

   Baseline finding: [add the subcategories that showed No Auditing in your Step 2 screenshot]

2. In Group Policy Management (gpmc.msc), created a new GPO called Lab-DC-Audit-Policy and linked it to the Domain Controllers OU. I left the Default Domain Controllers Policy untouched, so my changes are isolated, clearly named and easy to audit or roll back.

3. In the GPO, under Computer Configuration > Policies > Windows Settings > Security Settings > Advanced Audit Policy Configuration > Audit Policies, I configured:

   Account Logon
   - Audit Credential Validation: Success and Failure (4776 – NTLM credential validation)
   - Audit Kerberos Authentication Service: Success and Failure (4768 – Kerberos TGT requested; 4771 – Kerberos pre-authentication failed)

   Account Management
   - Audit User Account Management: Success and Failure (4720 – user created; 4724 – password reset attempted; 4740 – account locked out)
   - Audit Security Group Management: Success (4728 / 4732 – member added to a security group)

   Logon/Logoff
   - Audit Logon: Success and Failure (4624 – logon succeeded; 4625 – logon failed)
   - Audit Special Logon: Success (4672 – special privileges assigned to a new logon)

[Screenshot: lab17-02-advanced-audit-settings.png]

4. Under Security Settings > Local Policies > Security Options, defined and enabled "Audit: Force audit policy subcategory settings (Windows Vista or later) to override audit policy category settings", so the legacy category-level audit settings can't silently override the advanced ones.

5. Applied the policy and verified it:

```
gpupdate /force
```

   - gpupdate: apply Group Policy now instead of waiting for the next refresh
   - /force: reapply every policy, not just the ones that changed

```
auditpol /get /category:*
```

   The configured subcategories now showed the new values in place of the baseline settings.

[Screenshot: lab17-03-auditpol-after1.png]
[Screenshot: lab17-03-auditpol-after2.png]

Part B – Query and parse security events with PowerShell

1. Pulled the newest Kerberos TGT requests from the Security log:

```
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4768} -MaxEvents 5
```

   - Get-WinEvent: reads Windows event logs (the modern replacement for Get-EventLog)
   - -FilterHashtable: filters at the source, which is far faster than pulling the whole log and piping it to Where-Object
   - @{ }: a hashtable of key = value filters
   - LogName='Security': the Security log
   - Id=4768: Kerberos TGT requested
   - -MaxEvents 5: only the newest five

2. Read the full text of a successful logon event, to find the fields an analyst works from (Logon Type, Account Name, Source Network Address, Authentication Package):

```
(Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 1).Message
```

   - ( ): run the command inside first
   - .Message: show the event's full text rather than the one-line summary

3. Reviewed privileged logons:

```
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4672} -MaxEvents 5 | Format-List TimeCreated, Message
```

   - Id=4672: a logon that was assigned admin-level privileges
   - |: pass the results to the next command
   - Format-List TimeCreated, Message: show only those two properties, one per line

4. Built a reusable table of logon events. First I stored the events in a variable:

```
$events = Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 20
```

   - $events =: save the results in a variable so they can be reused without querying the log again

   Then I turned them into a table:

```
$events | Select-Object TimeCreated, @{n='Account';e={$_.Properties[5].Value}}, @{n='LogonType';e={$_.Properties[8].Value}}, @{n='SourceIP';e={$_.Properties[18].Value}} | Format-Table -AutoSize
```

   - Select-Object: choose which columns to show
   - @{n='...';e={...}}: a calculated column – n is the column name, e is the expression that fills it
   - $_: the current event in the pipeline
   - .Properties[5], [8], [18]: each event stores its data as a numbered list starting at 0. For event 4624, position 5 is the account name, 8 is the logon type and 18 is the source IP address.
   - Format-Table -AutoSize: display as a table with columns sized to fit

[Screenshot: lab17-04-logon-table.png]

What the table showed: the newest 20 logons were almost all DC01$ (the DC's own computer account, logon type 3) and SYSTEM (services, logon type 5), clustered around the time of a reboot. My own interactive logon was buried underneath them. That is the real-world picture: most 4624 events on a DC are machines and services, and part of an analyst's job is filtering that noise out so human logons stand out.


Key Learnings

Why put the settings in a new GPO instead of the Default Domain Controllers Policy?
Keeping the defaults untouched means my changes are isolated, clearly named and easy to review, report on or roll back. If something breaks, I can unlink one GPO instead of untangling edits to a built-in policy. It also makes the control easy to evidence in an audit.

Basic vs advanced audit policy
Windows has two audit systems. The older basic policy has nine broad categories. The advanced policy splits those into more than 50 subcategories, so you can log exactly what you need without flooding the log. If both are configured they can conflict, so the "force audit policy subcategory settings" option makes the advanced settings win. It is on by default, but defining it in the GPO guarantees it and documents the intent.

GPO "Not Configured" vs auditpol "No Auditing"
In the GPO editor, "Not Configured" means this GPO doesn't set that subcategory. It says nothing about what the machine is actually doing. auditpol /get shows the effective policy on the machine, which is the source of truth.

Logon types an analyst reads first (event 4624)
- 2 – Interactive: someone at the keyboard
- 3 – Network: something reached the machine over the network (SMB file share, remote PowerShell, tools like PsExec)
- 5 – Service: a service starting under an account
- 10 – RemoteInteractive: Remote Desktop (RDP)

Machine-account noise
Accounts ending in $ (such as DC01$) are computer accounts. Domain-joined machines authenticate constantly, especially after a reboot or a Group Policy refresh, so they dominate the Security log. Filtering them out is one of the first steps in any useful logon query or SIEM detection.

Triage scenario: SILKYLAB\Administrator, logon type 3, from a workstation's IP, at 2:14 am on a Sunday
- 4624 means the logon succeeded – this is access gained, not an attempt (a failure would be 4625).
- Type 3 means nobody sat at the DC or opened RDP. Something on that workstation reached the DC over the network using the admin credentials.
- Check the Authentication Package field. Inside a domain, Kerberos is expected. An admin network logon over NTLM is a classic indicator of pass-the-hash.
- Look for a 4672 (special privileges) for the same logon, then pivot to the workstation: what was running at 2:14 am, and is there a legitimate scheduled task or service that explains it?
- Out-of-hours admin activity from a workstation, with no change on record, should be treated as a live incident until it is explained.


Skills Demonstrated

- Baselining and verifying Windows audit policy with auditpol
- Designing and deploying an advanced audit policy GPO for domain controllers, following the practice of leaving default policies untouched
- Selecting audit subcategories and mapping them to key Security event IDs (4624, 4625, 4672, 4720, 4724, 4728, 4732, 4740, 4768, 4771, 4776)
- Querying the Windows Security log efficiently with Get-WinEvent and -FilterHashtable
- Parsing event data with PowerShell calculated properties to build reusable analyst views
- Recognising machine-account and service noise in authentication logs
- Triage reasoning for suspicious privileged logons (logon types, Kerberos vs NTLM, pass-the-hash indicators)
- Improving lab efficiency with VirtualBox Guest Additions and a shared clipboard
