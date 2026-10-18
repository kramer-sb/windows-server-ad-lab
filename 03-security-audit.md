# Part 3: The security angle

This is the part that makes the lab useful for SOC roles: generating events on purpose, then finding them.

## Turn on advanced audit policy

1. On DC01, create a GPO named `Audit-Policy` linked at the domain root (so it covers both the DC and the client).
2. Go to `Computer Configuration > Policies > Windows Settings > Security Settings > Advanced Audit Policy Configuration > Audit Policies`.
3. Enable, at minimum, Success and Failure on:
   - **Account Logon > Audit Credential Validation**
   - **Logon/Logoff > Audit Logon**
   - **Account Management > Audit User Account Management**
   - **Account Management > Audit Security Group Management**
   - Under **System > Audit Security State Change** (covers log clearing, event 1102, along with System audit policy generally)
4. Important: if `Computer Configuration > Security Options > Audit: Force audit policy subcategory settings to override audit policy category settings` isn't enabled, the legacy basic audit policy can silently override your advanced settings. Enable it in the same GPO.
5. `gpupdate /force` on both DC01 and the client.

## Generate the events

Do these from DC01 (or the client, where noted) so the DC's Security log captures them:

| Action | Event ID | Where |
| --- | --- | --- |
| Create a user | 4720 | DC01 Security log |
| Delete a user | 4726 | DC01 Security log |
| Add a user to a privileged group (e.g. Domain Admins) | 4728 (global group) / 4732 (local group) | DC01 Security log |
| Fail a logon (wrong password) | 4625 | DC01 or client Security log |
| Successful logon | 4624 | DC01 or client Security log |
| Clear a log | 1102 | The log that was cleared |

```powershell
# Create and delete a user
New-ADUser -Name "Temp Test" -SamAccountName tempuser -AccountPassword (Read-Host -AsSecureString) -Enabled $true
Remove-ADUser -Identity tempuser -Confirm:$false

# Add someone to a privileged group
Add-ADGroupMember -Identity "Domain Admins" -Members alice

# Fail a few logons: attempt an RDP or local logon with a wrong password a few times

# Clear a log (do this LAST, since it will also log a 1102 for itself)
wevtutil cl Security
```

## Find the events

6. Event Viewer: `Windows Logs > Security`, then filter by Event ID (right-click the log > Filter Current Log, enter `4624,4625,4720,4728,1102` in the Event IDs box).
7. Splunk: once the DC is forwarding logs to your Splunk instance (Universal Forwarder on DC01, or Windows Event Log input if Splunk is reading it directly), search:

```spl
index=* (EventCode=4624 OR EventCode=4625 OR EventCode=4720 OR EventCode=4728 OR EventCode=1102)
| table _time, EventCode, Account_Name, Target_Account, host
```

## Run Hayabusa

8. Export the Security log from DC01: in Event Viewer, right-click the Security log > Save All Events As > `security.evtx`.
9. Run Hayabusa against it, same as in SEC504:

```powershell
hayabusa.exe csv-timeline -f security.evtx -o hayabusa-results.csv
```

10. Open the CSV and compare Hayabusa's flagged detections against the events you generated — a 4728 (privileged group add) and a 1102 (log clear) should both show up as higher-severity hits, since those are classic post-compromise indicators.
