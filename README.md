# 🖥️ Windows 10 Client Setup & Active Directory Integration

![Windows 10](https://img.shields.io/badge/Windows%2010-Pro-blue?style=flat-square&logo=windows)
![Active Directory](https://img.shields.io/badge/Active%20Directory-Domain%20Joined-green?style=flat-square)
![RDP](https://img.shields.io/badge/RDP-Configured-green?style=flat-square)
![PowerShell](https://img.shields.io/badge/PowerShell-Automation-blue?style=flat-square&logo=powershell)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen?style=flat-square)

> **Home Lab Project — Part 2**
> Building on top of the DNS & DHCP setup from Part 1, this project documents the full deployment of a Windows 10 Pro client VM, joining it to an Active Directory domain, configuring RDP, and managing users and computers remotely.

---

## 📑 Table of Contents

- [Lab Environment](#lab-environment)
- [Objectives](#objectives)
- [Windows 10 VM Setup](#windows-10-vm-setup)
- [DNS Configuration](#dns-configuration)
- [Domain Join](#domain-join)
- [Domain User Creation](#domain-user-creation)
- [RDP Configuration](#rdp-configuration)
- [Computer Management in AD](#computer-management-in-ad)
- [Architecture](#architecture)
- [Issues & Fixes](#issues--fixes)
- [Skills Demonstrated](#skills-demonstrated)
- [Related Projects](#related-projects)

---

## 🏗️ Lab Environment

| Component | Details |
|---|---|
| **Hypervisor** | Oracle VirtualBox |
| **Domain Controller** | WIN-6ARV79FMB48 — Windows Server 2022 |
| **Client Machine** | CPC-NSimon-001 — Windows 10 Pro |
| **Domain** | mylab.local |
| **Server IP** | 192.168.1.200 |
| **Client IP** | 192.168.1.10 |
| **Network Adapter** | Bridged — RZ616 Wi-Fi 6E 160MHz |
| **Client RAM** | 2048 MB |
| **Client Storage** | 50 GB VDI |

---

## 🎯 Objectives

- [x] Create a Windows 10 Pro VM in VirtualBox
- [x] Install VirtualBox Guest Additions
- [x] Configure network adapter to Bridged mode
- [x] Set DNS to point to Domain Controller
- [x] Join Windows 10 VM to Active Directory domain
- [x] Verify domain join via PowerShell
- [x] Create a domain user account in Active Directory
- [x] Enable and configure RDP on Windows 10 client
- [x] RDP into Windows 10 using domain credentials
- [x] Verify computer object appears in Active Directory
- [x] Rename client computer via PowerShell
- [x] RDP using computer name instead of IP address

---

## 💻 Windows 10 VM Setup

### VirtualBox Settings

| Setting | Value |
|---|---|
| **Name** | OtherVMforPractice |
| **OS** | Windows 10 Pro (64-bit) |
| **RAM** | 2048 MB |
| **Storage** | 50 GB (VDI) |
| **Network** | Bridged Adapter |
| **ISO** | Windows.iso (4.56 GB) |

### Key Installation Decisions

**Why Windows 10 Pro and not Home?**
Windows 10 Home cannot join an Active Directory domain. Pro edition is required for enterprise domain membership — this is a critical distinction in real IT environments.

**Why "Set up for an organization"?**
During Windows setup, selecting "Set up for an organization" prepares the system for domain membership and enterprise management, which is the correct choice for any AD-joined machine.

### Guest Additions Installation
VirtualBox Guest Additions was installed to enable clipboard sharing between host and VM, making it easier to copy and paste PowerShell commands during configuration.

---

## 🌐 DNS Configuration

Before joining the domain, the Windows 10 client needed to use the Domain Controller as its DNS server. Without this, the client cannot resolve `mylab.local`.

### Problem
```
nslookup mylab.local
# *** UnKnown can't find mylab.local: No response from server
```

### Fix

```powershell
# Set DNS to Domain Controller
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("192.168.1.200")

# Flush DNS cache
Clear-DnsClientCache

# Verify DNS resolution
nslookup mylab.local 192.168.1.200
```

### Result ✅
```
Server:  winserver2022.homelab.local
Address: 192.168.1.200

Name:    mylab.local
Address: 192.168.1.200
```

---

## 🔗 Domain Join

With DNS correctly configured, the Windows 10 VM was joined to the Active Directory domain.

```powershell
# Join the domain
Add-Computer -DomainName "mylab.local" -Credential mylab.local\Administrator -Restart

# Verify after restart
systeminfo | findstr /B /C:"Domain"
```

### Result ✅
```
Domain: mylab.local
```

---

## 👤 Domain User Creation

A domain user account was created from the Domain Controller using PowerShell.

```powershell
New-ADUser -Name "Neil Marvin Simon" `
  -GivenName "Neil Marvin" `
  -Surname "Simon" `
  -SamAccountName "nsimon" `
  -UserPrincipalName "nsimon@mylab.local" `
  -AccountPassword (ConvertTo-SecureString "Welcome123!" -AsPlainText -Force) `
  -Enabled $true `
  -Path "CN=Users,DC=mylab,DC=local"
```

### Verification

```powershell
Get-ADUser -Identity "nsimon" | Select-Object Name, SamAccountName, UserPrincipalName, Enabled
```

### Result ✅

| Field | Value |
|---|---|
| **Name** | Neil Marvin Simon |
| **SamAccountName** | nsimon |
| **UserPrincipalName** | nsimon@mylab.local |
| **Enabled** | True |

---

## 🖥️ RDP Configuration

### Enable RDP on Windows 10 Client

```powershell
# Enable Remote Desktop
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' `
  -Name "fDenyTSConnections" -Value 0

# Open firewall for RDP on all profiles
Enable-NetFirewallRule -DisplayName "Remote Desktop*"
Set-NetFirewallRule -DisplayName "Remote Desktop*" -Profile Any -Enabled True

# Add domain user to Remote Desktop Users group
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "mylab\nsimon"

# Enable ping (ICMP)
Enable-NetFirewallRule -DisplayName "File and Printer Sharing (Echo Request - ICMPv4-In)"
```

### RDP Connection Methods

```powershell
# By IP address
mstsc /v:192.168.1.10

# By computer name (uses DNS resolution)
mstsc /v:CPC-NSimon-001

# By full domain name
mstsc /v:CPC-NSimon-001.mylab.local
```

**Login Credentials:**
- Username: `mylab\nsimon`
- Password: `Welcome123!`

> **Note:** RDP by computer name works because the DNS server resolves `CPC-NSimon-001` to `192.168.1.10` automatically — demonstrating DNS working in practice.

---

## 💾 Computer Management in Active Directory

### Verify Computer Objects

```powershell
Get-ADComputer -Filter * | Select-Object Name, DNSHostName, DistinguishedName
```

### Result ✅

| Name | DNS Hostname | Role |
|---|---|---|
| WIN-6ARV79FMB48 | WIN-6ARV79FMB48.mylab.local | Domain Controller |
| CPC-NSimon-001 | CPC-NSimon-001.mylab.local | Windows 10 Client |

### Rename Computer via PowerShell

The Windows 10 client was renamed from `DESKTOP-750KQC6` to `CPC-NSimon-001` following a professional naming convention.

```powershell
# Run on the Windows 10 VM directly
Rename-Computer -NewName "CPC-NSimon-001" -DomainCredential mylab\Administrator -Restart
```

**Naming Convention:**
- `CPC` = Client PC
- `NSimon` = User initials
- `001` = Device number

> **Important:** Computer names must be changed on the actual machine first. Active Directory cannot push name changes to computers — it only stores and syncs the information after the machine is renamed.

---

## 📐 Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    mylab.local Domain                    │
│                                                         │
│  ┌───────────────────────┐   ┌───────────────────────┐  │
│  │   WIN-6ARV79FMB48     │   │   CPC-NSimon-001      │  │
│  │   Windows Server 2022 │   │   Windows 10 Pro      │  │
│  │                       │   │                       │  │
│  │  ● Domain Controller  │◄──┤  ● Domain Joined      │  │
│  │  ● DNS Server         │   │  ● RDP Enabled        │  │
│  │  ● DHCP Server        │──►│  ● User: nsimon       │  │
│  │  ● AD DS              │   │                       │  │
│  │                       │   │                       │  │
│  │  IP: 192.168.1.200    │   │  IP: 192.168.1.10     │  │
│  └───────────────────────┘   └───────────────────────┘  │
│                                                         │
│         Network: 192.168.1.0/24                         │
│         Gateway: 192.168.1.1                            │
└─────────────────────────────────────────────────────────┘
```

---

## 🔴 Issues & Fixes

### Issue 1: mylab.local DNS Not Resolving
| | |
|---|---|
| **Error** | `*** UnKnown can't find mylab.local: No response from server` |
| **Cause** | Client was using home router's DNS (IPv6) instead of the DC |
| **Fix** | Manually set DNS to 192.168.1.200 using `Set-DnsClientServerAddress` |

### Issue 2: Domain Join — Access Denied
| | |
|---|---|
| **Error** | `Failed to join domain 'mylab.local' — Access is denied` |
| **Cause** | Wrong credential format and DNS not yet configured |
| **Fix** | Fixed DNS first, used `mylab.local\Administrator` format with correct password |

### Issue 3: Ping Timeout from Server to Client
| | |
|---|---|
| **Error** | `Request timed out` on all 4 packets |
| **Cause** | Windows 10 Firewall blocks ICMP by default |
| **Fix** | Enabled ICMPv4 Echo Request rule via `Enable-NetFirewallRule` |

### Issue 4: Remote Rename — RPC Unavailable
| | |
|---|---|
| **Error** | `The RPC server is unavailable (0x800706BA)` |
| **Cause** | WinRM not configured on client, blocking remote PowerShell |
| **Fix** | Ran `Rename-Computer` directly on the Windows 10 VM instead |

---

## 📚 Skills Demonstrated

- Windows 10 Pro Deployment in VirtualBox
- Active Directory Domain Join
- PowerShell Scripting & Remote Administration
- DNS Client Configuration
- Remote Desktop Protocol (RDP) Setup & Management
- Active Directory User Account Creation
- Active Directory Computer Object Management
- Computer Renaming via PowerShell
- Windows Firewall Rule Configuration
- Network Troubleshooting & Problem Solving
- IT Documentation

---

## 🚀 Future Improvements

- [ ] Configure Group Policy Objects (GPOs)
- [ ] Set up roaming profiles for domain users
- [ ] Implement password policies via GPO
- [ ] Configure folder redirection
- [ ] Set up additional client VMs
- [ ] Enable WinRM for full remote management
- [ ] Configure Windows Defender policies via GPO

---

## 🔗 Related Projects

| Project | Description |
|---|---|
| **Part 1** | [DNS & DHCP Server Setup](../part1-dns-dhcp/README.md) |
| **Part 2** | Windows 10 Client & AD Integration ← You are here |
| **Part 3** | Group Policy Objects (GPOs) — Coming Soon |

---

## 👤 Author

**Neil Marvin Simon**
Home Lab Project — Part 2 | Windows Server 2022 | Active Directory | Windows 10
*Built for IT Portfolio purposes*




