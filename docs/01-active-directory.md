# Active Directory Deployment and Administration

## Overview

Windows enterprise administration lab deployed on Proxmox VE using Windows Server 2022, Windows 11 Enterprise, and pfSense.

The environment supports Active Directory administration, centralized authentication, Group Policy, and IT support troubleshooting.

## Infrastructure

### Virtual Machines

| Machine  | Operating System      | Role                                       |
| -------- | --------------------- | ------------------------------------------ |
| DC01     | Windows Server 2022   | Active Directory Domain Controller and DNS |
| WIN11-01 | Windows 11 Enterprise | Domain-joined workstation                  |
| pfSense  | pfSense               | Gateway and DHCP                           |

### Network Configuration

| Setting              | Value           |
| -------------------- | --------------- |
| Network              | `10.0.0.0/24`   |
| Gateway              | `10.0.0.1`      |
| Domain Controller    | `DC01`          |
| Domain Controller IP | `10.0.0.100`    |
| AD Domain            | `corp.lab.test` |
| NetBIOS Name         | `CORP`          |
| Workstation          | `WIN11-01`      |

## Windows Server Deployment

Windows Server 2022 was deployed on Proxmox VE.

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

### Hostname Configuration

```powershell
Rename-Computer -NewName "DC01" -Restart
hostname
```

The server hostname was configured as `DC01`.

### Static IP Configuration

DC01 uses a static address to ensure that domain clients can consistently locate Active Directory and DNS services.

| Setting         | Value           |
| --------------- | --------------- |
| IP Address      | `10.0.0.100`    |
| Subnet Mask     | `255.255.255.0` |
| Default Gateway | `10.0.0.1`      |
| Preferred DNS   | `10.0.0.100`    |
| DHCP            | Disabled        |

```powershell
netsh interface ipv4 set address name="Ethernet" static 10.0.0.100 255.255.255.0 10.0.0.1

Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.0.100

ipconfig /all
```

### Active Directory and DNS Installation

Active Directory Domain Services and management tools were installed:

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

The Active Directory forest was created with integrated DNS:

```powershell
Install-ADDSForest -DomainName "corp.lab.test" -DomainNetbiosName "CORP" -InstallDNS
```

DC01 hosts the DNS zone for `corp.lab.test`. Active Directory uses DNS service records to help clients locate domain controllers and authentication services.

An upstream DNS forwarder can be configured for external resolution:

```powershell
Set-DnsServerForwarder -IPAddress 10.0.0.1
```

### Domain Verification

```powershell
Get-ADDomain | Select-Object DNSRoot,NetBIOSName,PDCEmulator

Get-Service NTDS,DNS

Resolve-DnsName -Type SRV _ldap._tcp.dc._msdcs.corp.lab.test
```

Verified results:

| Check           | Result               |
| --------------- | -------------------- |
| DNS Root        | `corp.lab.test`      |
| NetBIOS Name    | `CORP`               |
| PDC Emulator    | `DC01.corp.lab.test` |
| NTDS Service    | Running              |
| DNS Service     | Running              |
| LDAP SRV Target | `dc01.corp.lab.test` |
| LDAP Port       | 389                  |
| DC01 Address    | `10.0.0.100`         |

![Active Directory Domain Verification](../screenshots/ad-domain-verification.png)

## Windows 11 Enterprise Deployment

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
| Network Bridge   | vmbr1                 |
| Network Adapter  | Intel E1000           |

Windows 11 was installed from an ISO image. The 50 GiB virtual disk is below Microsoft's 64 GB minimum storage requirement and should be expanded when capacity permits.

### VirtIO and QEMU Guest Agent

VirtIO guest tools were installed, and the QEMU Guest Agent was enabled and tested from Proxmox:

```bash
qm set 113 --agent enabled=1
qm guest cmd 113 ping
```

### Domain Join

WIN11-01 was configured to use DC01 for DNS while retaining DHCP for IP addressing.

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.0.0.100
```

| Setting         | Value           |
| --------------- | --------------- |
| IPv4 Address    | `10.0.0.18`     |
| Subnet Mask     | `255.255.255.0` |
| Default Gateway | `10.0.0.1`      |
| DNS Server      | `10.0.0.100`    |
| DHCP            | Enabled         |

The workstation was joined using domain administrator credentials:

```powershell
Add-Computer -DomainName "corp.lab.test" -Credential (Get-Credential) -Restart
```

Domain membership was verified:

```powershell
Get-CimInstance Win32_ComputerSystem |
    Select-Object Name,Domain,PartOfDomain
```

```text
Name       Domain          PartOfDomain
----       ------          ------------
WIN11-01   corp.lab.test   True
```

### Domain Connectivity Verification

The initial `Test-ComputerSecureChannel` command returned `False`. Additional diagnostics found no trust failure.

```powershell
nltest /sc_verify:corp.lab.test
Test-ComputerSecureChannel -Verbose
```

Results:

```text
Trust Verification Status = 0 0x0 NERR_Success
True
```

![Domain Trust Verification](../screenshots/domain-trust-verification.png)

SMB connectivity was verified:

```powershell
Test-NetConnection 10.0.0.100 -Port 445
```

```text
TcpTestSucceeded : True
```

Time synchronization was checked:

```powershell
w32tm /query /status
```

```text
Source: DC01.corp.lab.test
Leap Indicator: 0 (no warning)
```

No secure-channel repair was required.

## Active Directory User Administration

An `Employees` organizational unit was created under `corp.lab.test` using Active Directory Users and Computers (`dsa.msc`).

A test employee account was created:

| Attribute           | Value                  |
| ------------------- | ---------------------- |
| Name                | John Smith             |
| Username            | `jsmith`               |
| User Principal Name | `jsmith@corp.lab.test` |
| Domain Logon        | `CORP\jsmith`          |
| Organizational Unit | Employees              |

The account was used to sign in to WIN11-01.

```powershell
whoami
```

```text
corp\jsmith
```

![Employee Authentication](../screenshots/employee-authentication.png)

---

## Exercise 01 — Employee Password Reset

### Scenario

John Smith forgot his Windows password and requested an account reset.

### Investigation

On DC01, account status was inspected using:

```powershell
Get-ADUser jsmith -Properties Enabled,LockedOut,PasswordExpired,PasswordLastSet
```

The account was enabled and not locked.

### Resolution

In Active Directory Users and Computers:

`corp.lab.test → Employees → John Smith → Reset Password`

A new password was assigned.

### Verification

```powershell
Get-ADUser jsmith -Properties Enabled,LockedOut,PasswordExpired,PasswordLastSet |
    Select-Object Name,Enabled,LockedOut,PasswordExpired,PasswordLastSet
```

```text
Name            : John Smith
Enabled         : True
LockedOut       : False
PasswordExpired : False
PasswordLastSet : 10/28/2026 2:23:07 PM
```

![Employee Password Reset Verification](../screenshots/employee-password-reset.png)

---

## Exercise 02 — Active Directory Account Lockout

### Scenario

Repeated incorrect password attempts caused John Smith's domain account to become locked.

### Lockout Policy Configuration

The Default Domain Policy was configured through Group Policy Management (`gpmc.msc`):

`Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Account Lockout Policy`

| Policy                        | Configuration            |
| ----------------------------- | ------------------------ |
| Lockout Threshold             | 3 invalid logon attempts |
| Lockout Duration              | 30 minutes               |
| Reset Counter After           | 30 minutes               |
| Administrator Account Lockout | Enabled                  |

![Domain Account Lockout Policy](../screenshots/domain-lockout-policy.png)

### Reproducing the Lockout

Incorrect passwords were entered for `CORP\jsmith` on WIN11-01.

Windows displayed:

```text
The referenced account is currently locked out and may not be logged on to.
```

![Employee Account Locked](../screenshots/account-lock.png)

### Investigation

ADUC identified the account as locked. The following PowerShell command can also inspect the account state:

```powershell
Get-ADUser jsmith -Properties LockedOut,BadPwdCount,LastBadPasswordAttempt |
    Select-Object Name,LockedOut,BadPwdCount,LastBadPasswordAttempt
```

On DC01, Event Viewer (`eventvwr.msc`) recorded the lockout:

| Attribute               | Value                 |
| ----------------------- | --------------------- |
| Event ID                | 4740                  |
| Account                 | `jsmith`              |
| Originating Workstation | `WIN11-01`            |
| Domain                  | `CORP`                |
| Timestamp               | 10/28/2026 5:22:09 PM |

![Event Viewer Account Lockout](../screenshots/event-viewer-lockout.png)

### Resolution and Verification

The account was unlocked through ADUC.

![Active Directory Account Unlock](../screenshots/account-unlock.png)

Equivalent PowerShell command:

```powershell
Unlock-ADAccount -Identity jsmith
```

John successfully signed in to WIN11-01 using the correct password after the unlock.

---

## Exercise 03 — Employee Account Deactivation and Reactivation

### Scenario

A simulated employee offboarding request required disabling John Smith's domain account. A subsequent reactivation request required restoring it.

### Account Deactivation

John's account was disabled using ADUC.

![Disable Employee Account](../screenshots/disable-account.png)

The state was verified on DC01:

```powershell
Get-ADUser jsmith | Select-Object Name,Enabled
```

```text
Name       Enabled
----       -------
John Smith False
```

![Disabled Account Verification](../screenshots/disable-account-validation-ps.png)

WIN11-01 initially still had a session under `CORP\jsmith`. A subsequent sign-in attempt was rejected:

```text
Your account has been disabled. Please see your system administrator.
```

![Disabled Account Sign-In](../screenshots/disable-account-validation.png)

An existing Windows session does not automatically terminate when an AD account is disabled.

### Security Event Verification

The domain controller's Security log recorded Event ID `4725`.

| Attribute      | Value                 |
| -------------- | --------------------- |
| Event ID       | 4725                  |
| Action         | User account disabled |
| Target Account | `CORP\jsmith`         |
| Administrator  | `CORP\Administrator`  |
| Timestamp      | 10/28/2026 4:15:50 PM |

![Account Disabled Security Event](../screenshots/disable-account-event-viewer.png)

### Account Reactivation

John's account was re-enabled using ADUC.

![Enable Employee Account](../screenshots/enable-account.png)

Equivalent PowerShell command:

```powershell
Enable-ADAccount -Identity jsmith
```

AD verification showed `Enabled: True`.

![Enabled Account Verification](../screenshots/enable-account-validation-ps.png)

The Security log also recorded the reactivation:

| Attribute      | Value                 |
| -------------- | --------------------- |
| Event ID       | 4722                  |
| Action         | User account enabled  |
| Target Account | `CORP\jsmith`         |
| Administrator  | `CORP\Administrator`  |
| Timestamp      | 10/28/2026 4:24:21 PM |

![Account Enabled Security Event](../screenshots/enable-account-event-viewer.png)

Account reactivation was verified in AD. A separate successful post-reactivation workstation sign-in was not captured.

---

## Exercise 04 — Active Directory Security Group Management

### Scenario

John Smith required membership in the IT department's security group to receive departmental resource permissions.

### Group Configuration

A `Groups` organizational unit was created under `corp.lab.test`.

A new group was configured:

| Attribute           | Value        |
| ------------------- | ------------ |
| Name                | `IT-Support` |
| Organizational Unit | Groups       |
| Scope               | Global       |
| Type                | Security     |

![IT Security Group](../screenshots/it-security-group.png)

### Membership Assignment

John Smith (`CORP\jsmith`) was added through the group's **Members** tab in ADUC.

![IT-Support Group Members](../screenshots/add-employee.png)

### Verification

On DC01:

```powershell
Get-ADPrincipalGroupMembership jsmith | Select-Object Name
```

The results included `Domain Users` and `IT-Support`.

![Security Group Membership Verification](../screenshots/add-employee-validation.png)

On WIN11-01, John's session token was checked:

```powershell
whoami /groups
```

The output included `CORP\IT-Support`, confirming the group membership was present in his Windows logon token.

---

## Exercise 05 — SMB Shared Folder and NTFS Permissions

### Scenario

An IT employee needed read/write access to a departmental network share. A permissions restriction was then introduced to simulate an Access Denied issue.

### Shared Folder and NTFS Configuration

The folder was created on DC01:

```text
C:\Shares\IT-Shared
```

NTFS inheritance was disabled, with permissions applying to the folder, subfolders, and files.

| Principal       | NTFS Permission |
| --------------- | --------------- |
| SYSTEM          | Full Control    |
| Administrators  | Full Control    |
| CORP\IT-Support | Modify          |

![NTFS Folder Permissions](../screenshots/ntfs-folder-permissions.png)

### SMB Share Configuration

The share was created using PowerShell:

```powershell
New-SmbShare -Name "IT-Shared" `
    -Path "C:\Shares\IT-Shared" `
    -ChangeAccess "CORP\IT-Support" `
    -FullAccess "BUILTIN\Administrators"
```

Verification:

```powershell
Get-SmbShareAccess -Name "IT-Shared"
```

| Principal              | SMB Permission |
| ---------------------- | -------------- |
| BUILTIN\Administrators | Full           |
| CORP\IT-Support        | Change         |

![SMB Share Permissions](../screenshots/smb-share-permissions.png)

The share was accessible at `\\DC01\IT-Shared`.

### Access Verification

On WIN11-01, signed in as `CORP\jsmith`:

```powershell
Test-NetConnection DC01 -Port 445
```

```text
TcpTestSucceeded : True
```

File read, modification, and deletion were tested:

```powershell
Get-Content "\\DC01\IT-Shared\test.txt"
Add-Content "\\DC01\IT-Shared\test.txt" -Value "Modified"
Get-Content "\\DC01\IT-Shared\test.txt"
Remove-Item "\\DC01\IT-Shared\test.txt"
```

The modified file contained:

```text
FileModified
```

![Shared Folder Access and Modification](../screenshots/shared-folder-access-and-modification.png)

### Simulating an Access Denied Issue

On DC01, the `CORP\IT-Support` NTFS permission was changed from **Modify** to **Read & Execute**. SMB permissions were not changed.

| Permission Layer | CORP\IT-Support |
| ---------------- | --------------- |
| SMB              | Change          |
| NTFS             | Read & Execute  |

![NTFS Permission Restrictions](../screenshots/smb-permissions-restrictions.png)

On WIN11-01, listing the share succeeded:

```powershell
Get-ChildItem "\\DC01\IT-Shared\"
```

Creating a file failed:

```powershell
Set-Content "\\DC01\IT-Shared\restricted-test.txt" -Value "Testing access"
```

```text
Set-Content : Access to the path '\\DC01\IT-Shared\restricted-test.txt' is denied.
```

The error was classified as `PermissionDenied` / `UnauthorizedAccessException`.

![SMB Restrictions Verification](../screenshots/smb-restrictions-validation.png)

### Root Cause

SMB Change access permitted modifications at the share level, but NTFS Read & Execute restricted write access. Network file operations are constrained by both permission layers.
