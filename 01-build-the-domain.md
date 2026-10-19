# Part 1: Build the domain

All of this runs on the Windows host, inside VirtualBox.

1. In VirtualBox, create a new VM (Windows Server 2022/2025, 4 GB RAM, 60 GB disk) and attach the Evaluation ISO. Install the Desktop Experience edition so you have a GUI.
2. Network adapter: set it to **Bridged Adapter** before first boot, pointed at your host's physical NIC.
3. After install, set a static IP from an elevated PowerShell prompt:

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.0.0.70 -PrefixLength 24 -DefaultGateway 10.0.0.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
```

(The gateway lets DC01 reach the internet for updates. Set DNS to 127.0.0.1 now; after the AD DS and DNS roles are installed and the forest is created, add a DNS forwarder in the later step below.)

4. Add the AD DS and DNS roles:

```powershell
Install-WindowsFeature AD-Domain-Services, DNS -IncludeManagementTools
```

5. Promote it to a new forest:

```powershell
Install-ADDSForest -DomainName "lab.local" -InstallDns
```

It will prompt for a Directory Services Restore Mode password, then reboot.

6. After reboot, log back in as `LAB\Administrator` and create a few OUs, users, and groups:

```powershell
New-ADOrganizationalUnit -Name "IT" -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Finance" -Path "DC=lab,DC=local"
New-ADGroup -Name "IT" -GroupScope Global -Path "OU=IT,DC=lab,DC=local"
New-ADGroup -Name "Finance" -GroupScope Global -Path "OU=Finance,DC=lab,DC=local"
New-ADUser -Name "Alice IT" -SamAccountName alice -Path "OU=IT,DC=lab,DC=local" -AccountPassword (Read-Host -AsSecureString "Password") -Enabled $true
Add-ADGroupMember -Identity IT -Members alice
```

Repeat for a Finance user.

Then add a forwarder so DC01 can still resolve everything outside `lab.local`, including your `.lab` names from CoreDNS at 10.0.0.30 and the internet:

```powershell
Add-DnsServerForwarder -IPAddress 10.0.0.30
```

The client will use only DC01 for DNS, so this forwarder is what keeps `.lab` names and internet lookups working from the client.

Now switch to Proxmox for the client:

7. Create or clone the Windows 11 VM. If you're cloning win11-intune-lab, use a full clone from the `clean-install-before-enroll` snapshot, then rename it to WIN11-CLIENT inside Windows (`Rename-Computer -NewName WIN11-CLIENT -Restart`) and confirm `dsregcmd /status` shows `AzureAdJoined : NO`. If you build a new VM instead, it needs **UEFI (OVMF)** firmware, an EFI disk, and a TPM state disk (vTPM 2.0) added in the VM's Hardware tab, or Windows 11 setup will refuse to install. Attach it to vmbr0.
8. Inside the VM, set a static IP and point DNS at the DC:

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.0.0.75 -PrefixLength 24 -DefaultGateway 10.0.0.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.0.70
```

9. Confirm it can resolve the domain before joining: `Resolve-DnsName lab.local`. If that fails, stop here and check the troubleshooting file before moving on.
10. Join the domain (Windows 11 Pro or Enterprise only; Home can't):

```powershell
Add-Computer -DomainName "lab.local" -Credential (Get-Credential) -Restart
```
