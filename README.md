# Windows Server AD Lab (split-host: VirtualBox + Proxmox)

A hands-on Active Directory lab covering domain setup, Group Policy, security event auditing with Hayabusa and Splunk, and a Mimikatz/Burp refresher — all run across two physical hosts to keep RAM usage sane (one Windows VM per host at a time).

## Layout

- [`00-overview.md`](00-overview.md) — topology, prerequisites, VM sizing
- [`01-build-the-domain.md`](01-build-the-domain.md) — Server install, AD DS/DNS, forest promotion, OUs/users/groups, client domain join
- [`02-group-policy.md`](02-group-policy.md) — GPO settings, gpupdate/gpresult, inheritance/enforcement/security filtering
- [`03-security-audit.md`](03-security-audit.md) — advanced audit policy, generating and finding security events, Hayabusa
- [`04-mimikatz-burp.md`](04-mimikatz-burp.md) — credential dumping and Burp Suite against lab targets
- [`05-troubleshooting.md`](05-troubleshooting.md) — cross-host networking checklist

## Lab topology

| VM | Host | Role | IP |
| --- | --- | --- | --- |
| DC01 | Windows host (VirtualBox) | Domain controller, DNS | 10.0.0.70 |
| WIN11-CLIENT | Proxmox | Domain-joined client | 10.0.0.75 |

Full details in [`00-overview.md`](00-overview.md).
