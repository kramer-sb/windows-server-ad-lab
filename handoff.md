# Handoff — 2026-10-19

## Where things stand

- DC01 (10.0.0.70, VirtualBox on Windows host): forest `lab.local` promoted, AD DS + DNS roles installed, DNS forwarder set to 10.0.0.30. OUs/users/groups from Part 1 not yet confirmed as created (check before Part 2).
- WIN11-CLIENT (10.0.0.75, Proxmox): static IP set, DNS pointed at 10.0.0.70, domain join (`Add-Computer -DomainName "lab.local"`) completed successfully using `LAB\Administrator`. Machine rebooted and is currently running Windows Update.
- Both VMs will be powered off for the night. Proxmox host itself stays running.

## Next time you sit down

1. Power both VMs back on. Let WIN11-CLIENT finish any pending updates before doing anything else.
2. Confirm the domain join actually stuck: on WIN11-CLIENT, log in as a domain account (`LAB\Administrator` or a domain user) and run `dsregcmd /status` or just check System Properties shows it's joined to `lab.local`, not a workgroup.
3. On DC01, confirm the OUs/users/groups from Part 1 step 6 exist (`Get-ADOrganizationalUnit -Filter *`, `Get-ADUser -Filter *`) — if that step got skipped or interrupted, finish it before moving to Group Policy.
4. Then move into **Part 2: Group Policy** (`02-group-policy.md` in the repo) — building the `IT-Policy` GPO, linking it, and testing with `gpupdate`/`gpresult`.

## Known gotchas from today (don't re-hit these)

- `Get-Credential` fails with a "missing mandatory parameter" error in non-interactive/limited consoles (e.g. some Proxmox console modes). Workaround: build the credential manually instead —
  ```powershell
  $cred = New-Object System.Management.Automation.PSCredential("lab\Administrator", (Read-Host -AsSecureString "Password"))
  ```
- `Add-Computer` (domain join) runs **on the client**, not on DC01. DC01 can't join its own forest.
- The lab's IP scheme is **10.0.0.0/24** (your real LAN), not an isolated range — DC01 is `.70`, WIN11-CLIENT is `.75`, gateway `.1`, and `.30` is an existing CoreDNS box the DC forwards to. `10.0.0.65` is already in use by the Wazuh VM, so don't reuse it.

## Repo

Everything's committed and pushed to `github.com/kramer-sb/windows-server-ad-lab`, main branch. Pull before you start if you touched the files from another machine.
