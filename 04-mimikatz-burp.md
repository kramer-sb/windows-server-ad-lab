# Part 4: Mimikatz and Burp refresher (lab only)

Run these only inside the isolated lab network. Don't run Mimikatz with AV/EDR active on a machine you care about; it will get flagged and quarantined immediately, which is expected and fine on a disposable lab VM, but not something to do on a production endpoint.

## Mimikatz: credential dumping against the lab client

1. On WIN11-CLIENT, disable or exclude the lab folder from Defender temporarily (lab only):

```powershell
Add-MpPreference -ExclusionPath "C:\Tools\mimikatz"
```

2. Run Mimikatz from an elevated prompt:

```
privilege::debug
sekurlsa::logonpasswords
```

This dumps credentials for currently logged-on sessions, which is why logging in as a domain user (e.g. `alice`) before running it matters — there needs to be something in LSASS to dump.

3. Note the timestamp, then go check DC01's logs for what this looked like from the domain's side: the logon that preceded the dump (4624), and if you also try `sekurlsa::pth` (pass-the-hash) to reuse a dumped hash, a 4624 with Logon Type 9 (NewCredentials) or 3 (Network), which is one of the classic pass-the-hash indicators analysts look for.
4. On the client itself, Sysmon (if installed) or Defender for Endpoint would normally flag `lsass.exe` being accessed by an unusual process — worth installing Sysmon with a basic config in this lab if you want that signal too.

## Burp Suite Community: a deliberately vulnerable target

5. Stand up a vulnerable app on its own lab VM — OWASP Juice Shop or DVWA are the standard choices and both run easily as a Docker container:

```bash
docker run -d -p 3000:3000 bkimminich/juice-shop
```

6. In Burp Community, set your browser's proxy to Burp (127.0.0.1:8080 by default), install Burp's CA cert in the browser so HTTPS traffic decrypts cleanly, and point the browser at the vulnerable app's address.
7. Use Proxy > HTTP history to capture requests as you walk through the app, then send an interesting request to Repeater to manipulate parameters (classic SQLi/IDOR testing) — Community Edition doesn't have the automated Scanner, so this part is manual, which is also closer to what SOC/analyst work actually looks like when triaging a report from a pentest.
