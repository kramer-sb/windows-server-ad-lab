# Handoff — 2026-10-20

## Where things stand

- Fixed timezone mismatch: DC01 was on Pacific, WIN11-CLIENT was also on Pacific; both now set to Eastern (`tzutil /s "Eastern Standard Time"`). DC01's time source is the local CMOS clock (no upstream NTP), which is fine for a single-DC lab but worth knowing if you ever add a second DC.
- Moved WIN11-CLIENT from the default `CN=Computers` container into the `IT` OU (GPOs can't link to containers, only OUs — this was blocking all of Part 2 testing until fixed).
- Confirmed domain logon and Group Policy processing work correctly for both computer and user side when testing as `LAB\Administrator`.
- Set up RDP access to WIN11-CLIENT for `alice` and `jess`, via a GPO (`IT-Policy`): enabled Remote Desktop, opened the Remote Desktop firewall rules, and added both accounts to the local Remote Desktop Users group through Group Policy Preferences (Local Users and Groups). Works for both, including jess (Finance OU) on a machine whose policy is scoped through the IT OU — a reminder that this specific setting is computer-scoped, not user-scoped.

## Part 2 (Group Policy) progress

- **Step 2 (password/lockout policy):** done on `Default Domain Policy`, as the tutorial specifies (account policies only take effect domain-wide regardless of where they're linked). Used NIST 800-63B-style settings instead of the tutorial's defaults: 16-char minimum with complexity, no forced expiration (max age 0), 10-password history, lockout after 3 attempts, 15-min duration, Administrator account lockout enabled.
- **Step 3 (ICMP firewall rule):** done, moved from an initial (wrong) placement on Default Domain Policy to `IT-Policy`, confirmed via `gpresult /r` after the OU move.
- **Step 4 (Control Panel restriction):** done, confirmed working for alice (blocked) and jess (not blocked, correctly out of scope). Hit a real gotcha here — see below.
- **Step 5 (AppLocker):** not started. This is where we left off.

## Known gotchas from today

- **The big one:** a Group Policy Preference (registry item) can appear to exist in the Group Policy Management Editor's tree without actually being committed to the GPO. Confirmed via `Get-GPO -Name "IT-Policy"` — `$gpo.User.DSVersion` / `$gpo.User.SysvolVersion` stayed at `0` even though the `NoControlPanel` registry item was visibly listed under User Configuration > Preferences > Windows Settings > Registry. Reopening the GPO editor showed the item was actually gone ("There are no items to show in this view"). Recreating it and confirming via the same `Get-GPO` version check (both numbers moved to `2` and matched) was what confirmed the real fix. **Lesson: don't trust that an item "is there" just because it's visible — verify with `Get-GPO`'s User/Computer DSVersion and SysvolVersion after closing the editor.**
- The "Prohibit access to Control Panel and PC settings" ADMX-backed policy (normally under `User Configuration > Policies > Administrative Templates > Control Panel`) did not appear in this Server 2025 GPMC's Control Panel node, with no filters active (Managed/Configured/Commented all at defaults, keyword and requirements filters both off). Worked around it with the registry preference equivalent (`HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer\NoControlPanel` = REG_DWORD 1) instead of chasing the missing ADMX setting further. If you ever need other Administrative Template settings that don't show up, this is a viable fallback pattern.
- Account/password-policy GPOs must go on `Default Domain Policy` (or another GPO linked at the domain root) — linking them to an OU-scoped GPO silently does nothing for domain accounts.

## Next time you sit down

1. Pick up at **Step 5: AppLocker**, in `IT-Policy`, Computer Configuration side.
   - Create default rules for all three collections (Executable, Windows Installer, Script) before anything else, or you risk locking out even administrators once enforcement is on.
   - Add one explicit rule (e.g. block `cmd.exe` for standard users in the IT OU) to have something concrete to test.
   - Enable the AppLocker service on the client: `Set-Service -Name AppIDSvc -StartupType Automatic; Start-Service AppIDSvc`.
2. After Step 5, steps 8–11 cover inheritance, enforced links, and security filtering — the part of the tutorial that's most relevant to real-world GPO troubleshooting, and we already have a live example of OU-based scoping working correctly (alice vs. jess on Control Panel) to build on.

---

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

## Commands 

`nltest /dsgetdc:lab.local` - confirm client locates the domain controller

`nltest /sc_verify:lab.local` - verifies secure channel

`gpresult /r` - confirm Grup Policy is applying
