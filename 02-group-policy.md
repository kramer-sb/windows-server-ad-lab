# Part 2: Practice Group Policy

Do this from DC01, using Group Policy Management (`gpmc.msc`).

## Build a GPO with several settings

1. Create a GPO named `IT-Policy` and link it to the `IT` OU.
2. Password and account lockout policy (these live under Computer Configuration, so they apply at the domain or an OU containing computer objects — for a lab, set them in `Default Domain Policy` or create a dedicated computer-side GPO linked at the domain root):
   `Computer Configuration > Policies > Windows Settings > Security Settings > Account Policies`
   - Password Policy: minimum length 12, complexity enabled, max age 60 days.
   - Account Lockout Policy: threshold 5 invalid attempts, lockout duration 15 minutes.
3. Firewall rule, in the same GPO:
   `Computer Configuration > Policies > Windows Settings > Security Settings > Windows Firewall with Advanced Security`
   - New inbound rule allowing ICMPv4 (Echo Request), so you can ping domain members for testing.
4. Control Panel restriction, under User Configuration (so link this part to the `IT` OU, which should contain your user accounts, not just computers):
   `User Configuration > Policies > Administrative Templates > Control Panel`
   - Enable "Prohibit access to Control Panel and PC settings."
5. AppLocker rule:
   `Computer Configuration > Policies > Windows Settings > Security Settings > Application Control Policies > AppLocker`
   - Right-click each rule collection (Executable, Windows Installer, Script) and choose "Create Default Rules" first, so you don't lock yourself out.
   - Then add one explicit rule, e.g. block `cmd.exe` for the IT OU's standard users, to see AppLocker actually deny something.
   - AppLocker also needs the Application Identity service running on the client: `Set-Service -Name AppIDSvc -StartupType Automatic; Start-Service AppIDSvc`.

## Apply and verify on the client

6. On WIN11-CLIENT:

```powershell
gpupdate /force
gpresult /r
```

`gpresult /r` shows which GPOs applied and which were filtered out, which is the fastest way to confirm a link, a security filter, or an OU placement did what you expected.

7. For a full picture, including the actual registry/security settings a GPO wrote: `gpresult /h report.html` and open the report.

## Test inheritance, enforced links, and security filtering

This is the part most people skip, and it's where most real-world GPO troubleshooting actually happens.

8. Create a second OU under `IT`, e.g. `IT\Contractors`, and move a test user into it. By default it inherits everything linked to `IT` above it — confirm with `gpresult /r` as that user.
9. Link a second GPO directly to `IT\Contractors` that explicitly disables one of the settings from `IT-Policy` (e.g. a less strict Control Panel setting). Without "Enforced," the child OU's GPO wins over the parent's conflicting setting — that's normal precedence, not a bug.
10. Go back to `IT-Policy`, right-click it in the GPMC console tree under the `IT` link, and select **Enforced**. Re-run `gpupdate /force` on the client and confirm with `gpresult /r` that `IT-Policy` now wins over the child OU's GPO, because Enforced overrides normal child-OU precedence.
11. Security filtering: on `IT-Policy`'s Scope tab, remove "Authenticated Users" from Security Filtering and add only the `IT` security group. Move your test user out of the `IT` group (but leave them in the `IT` OU) and confirm with `gpresult /r` that the GPO no longer applies to them, even though they're still in the OU it's linked to — OU membership and group membership are evaluated separately.
