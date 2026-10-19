# AD Lab Tutorial: Proxmox + VirtualBox Split-Host Setup

This lab splits across your two existing physical hosts instead of running both Windows machines on Proxmox at once. The Windows Server domain controller runs in VirtualBox on your Windows host. The Windows 11 client runs as a VM on Proxmox. Both land in the same IP subnet so they can reach each other over your existing switch, with the DC acting as DNS for the client.

Splitting it this way means Proxmox only ever carries one Windows VM (the client), keeping the RAM load on Proxmox to whatever that one VM needs, while the DC's RAM comes out of the Windows host instead.

## Lab topology and prerequisites

Both hosts sit on your existing LAN, so no VLAN or managed switch is required. Both VMs use static addresses from the reserved static range (10.0.0.2 to 10.0.0.100) on your existing **10.0.0.0/24** network, so they stay reachable from your other lab machines and keep internet access through the Xfinity gateway at 10.0.0.1. 10.0.0.65 is already taken by the Wazuh VM, so DC01 and WIN11-CLIENT take the next two free slots in the usual increments of 5.

| VM | Host | Role | IP |
| --- | --- | --- | --- |
| DC01 | Windows host (VirtualBox) | Domain controller, DNS | 10.0.0.70 |
| WIN11-CLIENT | Proxmox | Domain-joined client | 10.0.0.75 |

```
 Windows host (VirtualBox)                         Proxmox host
 ┌───────────────────────┐                    ┌───────────────────────┐
 │   DC01                │   same LAN switch  │   WIN11-CLIENT        │
 │   AD DS + DNS         │◄──────────────────►│   Domain-joined       │
 │   10.0.0.70           │  10.0.0.0/24       │   10.0.0.75           │
 └───────────────────────┘                    └───────────────────────┘
        ▲                                                │
        └──────────── client's DNS points here ──────────┘
```

DC01's network adapter in VirtualBox should be set to **Bridged Adapter** (not NAT or host-only), pointed at your host's physical NIC, so it gets a real address on the switch instead of being hidden behind VirtualBox's NAT. On Proxmox, attach WIN11-CLIENT to your default bridge (usually vmbr0), the same one tied to the host's physical NIC, so it lands on the same segment.

Things to have ready before starting:

1. A Windows Server 2022 or 2025 Evaluation ISO, downloaded from the Microsoft Evaluation Center (180-day eval, no license key needed).
2. A Windows 11 Pro or Enterprise machine. If you already have win11-intune-lab (VM 203), do a full clone from its `clean-install-before-enroll` snapshot instead of downloading another ISO, then rename the clone to WIN11-CLIENT. Don't clone from a later snapshot, since those carry the Entra ID and Intune enrollment. Home edition can't join a domain.
3. VirtualBox Guest Additions (for the DC VM, to make the console usable).
4. On Proxmox, UEFI firmware (OVMF) and a vTPM enabled on the Windows 11 VM; Windows 11 won't install without both.

Minimum/recommended sizing, since I don't know how much RAM is free on either host:

| VM | RAM (min / recommended) | vCPU | Disk |
| --- | --- | --- | --- |
| DC01 (Server, VirtualBox) | 2 GB / 4 GB | 2 | 60 GB |
| WIN11-CLIENT (Proxmox) | 4 GB / 8 GB | 2 | 64 GB |

Check free RAM on the Windows host (Task Manager > Performance) and on Proxmox (Datacenter > Summary) before committing to the recommended numbers; drop to the minimums if either host is tight.
