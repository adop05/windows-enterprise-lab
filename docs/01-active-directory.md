# 01 — Active Directory Deployment and Administration

## Overview

This document covers the deployment and configuration of a Windows enterprise environment using Windows Server 2022 and Windows 11 Enterprise.

## 1. Infrastructure

### Virtual Machines

| Machine  | Operating System      | Role               |
| -------- | --------------------- | ------------------ |
| DC01     | Windows Server 2022   | Domain Controller  |
| WIN11-01 | Windows 11 Enterprise | Domain Workstation |
| pfSense  | pfSense               | Gateway and DHCP   |

### Network Configuration

| Setting                    | Value         |
| -------------------------- | ------------- |
| Network                    | 10.0.0.0/24   |
| Gateway                    | 10.0.0.1      |
| Domain Controller IP       | 10.0.0.100    |
| Domain Controller Hostname | DC01          |
| Target Domain              | corp.lab.test |
| NetBIOS Domain             | CORP          |
| Workstation                | WIN11-01      |

## 2. Windows Server Deployment

Windows Server 2022 was deployed as a virtual machine within Proxmox.

### VM Configuration

| Component       | Configuration |
| --------------- | ------------- |
| Hypervisor      | Proxmox VE    |
| Firmware        | OVMF (UEFI)   |
| Machine         | q35           |
| CPU             | 4 vCPUs       |
| Memory          | 4 GiB         |
| Storage         | 40 GiB        |
| Network Adapter | VirtIO        |
| Network Bridge  | vmbr1         |
| TPM             | 2.0           |

## 3. Configuring the Server Hostname

The Windows Server hostname was changed to `DC01`.

```powershell
Rename-Computer -NewName "DC01" -Restart
```

After restarting, the hostname was verified:

```powershell
hostname
```

Expected output:

```text
DC01
```

This establishes a recognizable server identity for the domain controller.

## 4. Configuring Static IP Addressing

### Static IP Configuration

A domain controller should maintain a consistent network address so clients can reliably locate Active Directory DNS and authentication services.

DC01 was configured with the following settings:

| Setting         | Value         |
| --------------- | ------------- |
| IP Address      | 10.0.0.100    |
| Subnet Mask     | 255.255.255.0 |
| Default Gateway | 10.0.0.1      |
| Preferred DNS   | 10.0.0.100    |
| DHCP            | Disabled      |

The network address was configured using:

```powershell
netsh interface ipv4 set address name="Ethernet" static 10.0.0.100 255.255.255.0 10.0.0.1
```

The DNS client configuration was updated:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.0.100
```

Configuration was verified using:

```powershell
ipconfig /all
```

The output confirmed that DC01 had the expected hostname, static address, default gateway, and DNS configuration.

## 5. Active Directory DNS

Active Directory depends on DNS to locate domain controllers and authentication services.

Domain members use DNS service records to locate resources.

For example:

```text
_ldap._tcp.dc._msdcs.corp.lab.test
```

This DNS SRV record allows clients to discover domain controllers.

### **\_\_**

DC01 is intended to host the DNS zone for `corp.lab.test`.

Its DNS client points to `10.0.0.100` so the server can query its own Active Directory DNS service.

Public DNS requests can be forwarded to pfSense at `10.0.0.1`.

## 6. Installing Active Directory Domain Services

The following deployment commands should be verified against the current server state before execution.

### Check Existing Installation

```powershell
Get-WindowsFeature AD-Domain-Services
```

Check for existing services:

```powershell
Get-Service NTDS,DNS -ErrorAction SilentlyContinue
```

### Install AD DS

If not already installed:

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

This installs Active Directory Domain Services and its administrative tools.

### Create the Active Directory Forest

If the server has not already been promoted:

```powershell
Install-ADDSForest -DomainName "corp.lab.test" -DomainNetbiosName "CORP" -InstallDNS
```

This creates the Active Directory forest and configures DNS.

## 7. Verifying Active Directory

After promotion, verify the domain:

```powershell
Get-ADDomain
```

Check services:

```powershell
Get-Service NTDS,DNS
```

Expected status:

```text
NTDS    Running
DNS     Running
```

Verify DC01's DNS record:

```powershell
Resolve-DnsName dc01.corp.lab.test
```

Verify Active Directory SRV records:

```powershell
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.corp.lab.test
```

Configure pfSense as an upstream DNS forwarder if needed:

```powershell
Set-DnsServerForwarder -IPAddress 10.0.0.1
```

Test external DNS resolution:

```powershell
Resolve-DnsName microsoft.com
```

## 8. Windows 11 Enterprise Deployment

### VM Configuration

| Component        | Configuration         |
| ---------------- | --------------------- |
| Operating System | Windows 11 Enterprise |
| Firmware         | OVMF (UEFI)           |
| Machine          | q35                   |
| CPU              | 2 vCPUs               |
| Memory           | 4 GiB                 |
| Virtual Disk     | 50 GiB SATA           |
| TPM              | 2.0                   |
| Network          | vmbr1                 |
| Network Adapter  | Intel E1000           |

Windows 11 was installed using an ISO image.

The virtual machine uses a 50 GiB disk, which is below Windows 11's commonly specified 64 GB minimum storage requirement. The virtual disk should be expanded when sufficient host storage becomes available.

### VirtIO and QEMU Guest Agent

After installing Windows, VirtIO guest tools were installed.

The QEMU Guest Agent was enabled in Proxmox:

```bash
qm set 113 --agent enabled=1
```

Communication was tested from the Proxmox host:

```bash
qm guest cmd 113 ping
```

## 9. Joining Windows 11 to Active Directory

### Configure DNS

On WIN11-01:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.0.100
```

Verify network configuration:

```powershell
ipconfig /all
```

Test DNS resolution:

```powershell
Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.corp.lab.test
```

Test LDAP connectivity:

```powershell
Test-NetConnection 10.0.0.100 -Port 389
```

### Join the Domain

```powershell
Add-Computer -DomainName "corp.lab.test" -Credential (Get-Credential) -Restart
```

Use an account authorized to join computers to the domain.

After restarting, verify:

```powershell
Get-CimInstance Win32_ComputerSystem | Select-Object Name,Domain,PartOfDomain
```

Then:

```powershell
Test-ComputerSecureChannel
```

Expected:

```text
True
```

This verifies that the workstation's secure channel to the domain is functioning.
