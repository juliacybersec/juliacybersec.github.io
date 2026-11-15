Command Reference: PowerShell and CMD

A running list of every command I've used in my home lab, with what each part does and when I'd reach for it on the job. Updated at the end of each lab.

Shell key: PS = PowerShell only. CMD/PS = works in both Command Prompt and PowerShell.

---

NETWORKING AND DNS

ipconfig   (CMD/PS)   Used in: Lab 16

```
ipconfig
```

What it does: shows the IP settings for each network adapter (IPv4 address, subnet mask, default gateway).

Each part:
- ipconfig: the command itself. With no switches it gives the short summary.

On the job: the first thing to check when a PC "has no internet" or can't reach a server. In Lab 16 I used it to read DC01's static IP before pointing Win11's DNS at it.

Related switches to add as I use them: /all (adds DNS servers, DHCP and MAC address), /flushdns (clears the local DNS cache).

---

nslookup   (CMD/PS)   Used in: Lab 16

```
nslookup silkylab.local
```

What it does: asks a DNS server to look up a name and shows the IP address it returns.

Each part:
- nslookup: the DNS lookup tool.
- silkylab.local: the name to look up. Here it's my lab domain, which should return the Domain Controller's IP.

On the job: confirms that DNS is working and pointing at the right server. If this fails, a PC can't join or log in to the domain.

---

nslookup -type=SRV   (CMD/PS)   Used in: Lab 16

```
nslookup -type=SRV _ldap._tcp.dc._msdcs.silkylab.local
```

What it does: looks up the SRV (service) record that tells PCs where the Domain Controllers for a domain are.

Each part:
- nslookup: the DNS lookup tool.
- -type=SRV: asks for SRV records instead of the default A (address) records.
- _ldap._tcp.dc._msdcs.silkylab.local: the record name. Read it as "the LDAP service, over TCP, for Domain Controllers, in silkylab.local".

On the job: the go-to check for the error "An Active Directory Domain Controller for the domain could not be contacted". If this returns the DC's hostname, DNS is fine and the problem is somewhere else.

---

ACTIVE DIRECTORY (PowerShell, on a Domain Controller or a PC with RSAT installed)

Get-ADDomain   (PS)   Used in: Lab 16

```
Get-ADDomain
```

What it does: shows the details of the current AD domain: its DNS name, NetBIOS name, forest, and which DCs hold key roles.

Each part:
- Get-ADDomain: the cmdlet. With no parameters it reports on the domain the computer belongs to.

On the job: a quick confirmation that a domain exists and is healthy, and of what it's called.

---

Get-ADGroupMember   (PS)   Used in: Lab 16

```
Get-ADGroupMember -Identity "Lab-Helpdesk"
```

What it does: lists the members of an AD group.

Each part:
- Get-ADGroupMember: the cmdlet.
- -Identity: tells it which group to look up.
- "Lab-Helpdesk": the group name. The quotes keep it together as one value, which matters for names with spaces.

On the job: answers "Is this user in the group?" in seconds, without clicking through Active Directory Users and Computers.

---

Get-ADComputer   (PS)   Used in: Lab 16

```
Get-ADComputer -Filter *
```

What it does: lists computer accounts in AD.

Each part:
- Get-ADComputer: the cmdlet.
- -Filter: required when you don't use -Identity. Decides which computers to return.
- *: a wildcard meaning "everything", so every computer account is listed.

On the job: confirms that a PC actually joined the domain, and checks whether a computer account exists or is disabled.

---

IDENTITY AND LOGON

whoami   (CMD/PS)   Used in: Lab 16

```
whoami
```

What it does: shows the account you're logged in as, in the form domain\username.

Each part:
- whoami: the command itself.

On the job: proves whether a user is on a domain account (silkylab\tuser) or a local one (computername\user).

---

whoami /groups   (CMD/PS)   Used in: Lab 16

```
whoami /groups
```

What it does: lists every group in your current logon token.

Each part:
- whoami: shows who you're logged in as.
- /groups: also lists all your group memberships, including the built-in ones Windows adds automatically (Everyone, Authenticated Users, and so on).

On the job: when a user says "I was added to the group but still can't get in", this shows whether their logon picked it up. If the group is missing, they need to log off and back on to get a fresh token.

---

COMING NEXT

Group Policy (bonus lab): gpupdate /force, gpresult /r

Earlier labs to back-fill: BitLocker (manage-bde), mapped drives (net use), DNS troubleshooting
