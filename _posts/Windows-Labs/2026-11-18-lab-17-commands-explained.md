Commands Explained – Lab 17: Advanced Audit Policy

These are the commands from Lab 17, all run on the domain controller (DC01) in Terminal as Administrator. Each one shows the exact command and what every flag or argument does.


Audit policy (Command Prompt or PowerShell)

```
auditpol /get /category:*
```
What it does: shows the audit policy the machine is actually using, for every category and subcategory (Success, Failure, Success and Failure, or No Auditing). This is the source of truth. A GPO only shows what that one GPO sets.
- auditpol: Windows' built-in tool for viewing and setting audit policy
- /get: display the current settings
- /category:*: every category (* means all)

```
gpupdate /force
```
What it does: applies Group Policy now, so the new audit GPO takes effect straight away.
- gpupdate: refresh Group Policy
- /force: reapply every policy, not just the ones that changed


Reading the Security log (PowerShell)

```
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4768} -MaxEvents 5
```
What it does: lists the five newest Kerberos TGT requests.
- Get-WinEvent: reads Windows event logs (the modern replacement for Get-EventLog)
- -FilterHashtable: filters at the source, so only matching events are read. Piping the whole log to Where-Object reads everything first and is much slower.
- @{ }: a hashtable, a set of key = value filters separated by semicolons
- LogName='Security': which log to search
- Id=4768: which event ID to return
- -MaxEvents 5: return only the newest five

```
(Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 1).Message
```
What it does: shows the full text of the newest successful logon event, including Logon Type, Account Name, Source Network Address and Authentication Package.
- ( ): run the command inside the brackets first
- .Message: read the event's full message text instead of the one-line summary
- Id=4624: successful logon

```
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4672} -MaxEvents 5 | Format-List TimeCreated, Message
```
What it does: lists the five newest logons that were given admin-level privileges.
- Id=4672: special privileges assigned to a new logon
- |: pass the results to the next command
- Format-List TimeCreated, Message: show only these two properties, one per line

```
$events = Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 20
```
What it does: saves the 20 newest successful logons in a variable, so they can be reused without querying the log again.
- $events: a variable (variables in PowerShell start with $)
- =: store the result on the right in the variable on the left

```
$events.Count
```
What it does: shows how many events are stored in the variable. A quick check that the previous command worked.
- .Count: the number of items

```
$events | Select-Object TimeCreated, @{n='Account';e={$_.Properties[5].Value}}, @{n='LogonType';e={$_.Properties[8].Value}}, @{n='SourceIP';e={$_.Properties[18].Value}} | Format-Table -AutoSize
```
What it does: turns the stored logon events into a clean table showing when, who, how and from where.
- Select-Object: choose which columns to show
- TimeCreated: when the event was logged
- @{n='Account';e={...}}: a calculated column. n is the column name and e is the expression that fills it.
- $_: the current event as it passes through the pipeline
- .Properties[5].Value: the account name (event data is a numbered list starting at 0)
- .Properties[8].Value: the logon type (2 interactive, 3 network, 5 service, 10 RDP)
- .Properties[18].Value: the source IP address
- Format-Table -AutoSize: display as a table with columns sized to fit

Tip: for event 4624, .Properties[10].Value is the Authentication Package (Kerberos or NTLM). Add it as a fourth calculated column when hunting for NTLM admin logons.
