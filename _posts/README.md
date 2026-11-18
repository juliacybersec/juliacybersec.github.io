# audit-the-domain

Hands-on Windows security auditing: DC, advanced audit policy via GPO,
and reusable PowerShell detection queries for SOC-relevant events.

## What's in here

- `lab/` — VM configs, GPO XML exports, audit policy settings
- `queries/` — PowerShell scripts for pulling and parsing security events
- `notes/` — event field references (logon types, failure codes)

## How to follow along

1. Spin up a Windows Server DC (Hyper-V / VMware)
2. Apply the GPO in `lab/audit-gpo.xml`
3. Run the queries in `queries/` against the Security log

## Full write-up

[Link to blog post]   
