# Troubleshooting: VirtualBox Crash During Active Directory Domain Controller Promotion

## 1. Overview

This document describes the investigation and verification process following a VirtualBox crash during the promotion of a Windows Server 2025 virtual machine to an Active Directory Domain Controller.

The main objective was to determine whether the Active Directory Domain Services (AD DS) installation and domain controller promotion had completed successfully despite the VirtualBox crash.

### Environment

| Component | Configuration |
|---|---|
| Host OS | Windows 11 |
| CPU | AMD Ryzen AI 7 350 |
| Host Memory | 32 GB |
| Virtualization Software | Oracle VirtualBox 7.2.8 |
| Guest OS | Windows Server 2025 |
| Virtual Machine Name | DC01 |
| Domain Name | corp.test |
| NetBIOS Name | CORP |
| Virtual Machine Memory | 4096 MB |
| Virtual Machine CPUs | 2 |

---

## 2. Problem Description

During the Active Directory Domain Services (AD DS) installation and Domain Controller promotion process, VirtualBox unexpectedly crashed.

The virtual machine status changed to:

`Aborted`

The following error was displayed:

```text
The instruction at 0x00007ff8b8d3ee95
referenced memory at 0x000001db1afee000.
The memory could not be written.
```

### Initial Error Information

| Item | Value |
|---|---|
| Exception Code | 0xc0000005 |
| Exception Type | Access Violation |
| Faulting Module | VBoxVMM.DLL |
| Virtual Machine | DC01 |
| VirtualBox Status | Aborted |

The error indicated an access violation in the VirtualBox Virtual Machine Monitor component.

At this stage, it was unclear whether the AD DS promotion had completed successfully.

---

## 3. Investigation

### 3.1 VirtualBox Crash Log Analysis

The VirtualBox log file was located at:

```text
%USERPROFILE%\VirtualBox VMs\DC01\Logs\VBox.log
```

The following information was identified in the log:

```text
ExceptionCode=0xc0000005
ExceptionAddress=00007ff8b8d3ee95
```

The faulting module was identified as:

```text
VBoxVMM.DLL
```

This indicated that the crash occurred within the VirtualBox host-side virtualization component.

**Finding:**

The crash was associated with VirtualBox rather than being direct evidence of an Active Directory error.

### 3.2 Windows Hypervisor Environment

The following PowerShell command was executed:

```powershell
systeminfo | findstr /i "Hyper-V"
```

Result:

```text
Hyper-V Requirements:
A hypervisor has been detected.
```

This indicated that a Windows hypervisor was present on the host.

### 3.3 Windows Optional Features

The following command was used to inspect relevant Windows virtualization features:

```powershell
Get-WindowsOptionalFeature -Online |
Where-Object {$_.FeatureName -match 'Hyper-V|VirtualMachinePlatform|HypervisorPlatform|Containers-DisposableClientVM'} |
Select-Object FeatureName, State
```

Observed results:

| Feature | State |
|---|---|
| VirtualMachinePlatform | Enabled |
| HypervisorPlatform | Disabled |

### 3.4 Virtualization-Based Security (VBS)

The following command was executed:

```powershell
Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard |
Select-Object VirtualizationBasedSecurityStatus, SecurityServicesConfigured, SecurityServicesRunning
```

Observed results:

```text
VirtualizationBasedSecurityStatus : 2
SecurityServicesConfigured       : {2, 3, 4}
SecurityServicesRunning          : {2, 3, 4}
```

The results indicated that Virtualization-Based Security was running.

Memory Integrity was also confirmed as enabled in Windows Security.

### 3.5 VirtualBox NEM Investigation

The VirtualBox log contained the following information:

```text
HM: HMR3Init: Attempting fall back to NEM:
AMD-V is not available

NEM: WHvCapabilityCodeHypervisorPresent is TRUE
NEM: HypervisorPresent: 1

NEMR3Init: Snail execution mode is active!
```

**Finding:**

VirtualBox could not use AMD-V directly and attempted to use its NEM fallback execution path.

This suggested a possible interaction between the Windows hypervisor environment and VirtualBox.

However, this finding did not conclusively establish the root cause of the crash.

---

## 4. Troubleshooting and Recovery

### 4.1 Virtual Machine Configuration Review

The DC01 virtual machine settings were reviewed.

| Setting | Value |
|---|---|
| CPU Allocation | 2 CPUs |
| Execution Cap | 100% |
| Base Memory | 4096 MB |
| Hardware Virtualization | Enabled |
| Paravirtualization Interface | Default |
| Hard Disk Boot Order | Enabled |

No changes were made to these settings during this review.

### 4.2 Host System Restart

The Windows 11 host computer was restarted.

**Purpose:**

To reset temporary host operating system and virtualization runtime states and determine whether the VirtualBox crash would recur.

VBS and Memory Integrity were not disabled.

### 4.3 Virtual Machine Startup Verification

After restarting the host:

1. Opened Oracle VirtualBox.
2. Selected the DC01 virtual machine.
3. Started the virtual machine.
4. Waited for Windows Server to load.

**Result:**

Windows Server 2025 successfully reached the login screen.

The virtual machine did not immediately enter the Aborted state.

---

## 5. Active Directory Verification

After successfully starting Windows Server, several checks were performed to determine whether the AD DS promotion had completed successfully.

### 5.1 AD DS Service Verification

Command:

```powershell
Get-Service NTDS
```

Expected result:

```text
Status : Running
```

**Result:** Running

**Finding:** The Active Directory Domain Services service was running.

### 5.2 Domain Information Verification

Command:

```powershell
Get-ADDomain
```

The following values were confirmed:

| Attribute | Expected Value |
|---|---|
| DNSRoot | corp.test |
| NetBIOSName | CORP |

**Finding:** The domain information was successfully retrieved.

### 5.3 DNS Service Verification

Command:

```powershell
Get-Service DNS
```

**Result:** Running

The DNS service was running.

### 5.4 DNS Diagnostic Test

Command:

```powershell
dcdiag /test:DNS /v
```

**Result:** The summary of DNS test results reported PASS.

**Finding:** The DNS diagnostic test passed.

### 5.5 Domain Controller Advertising Test

Command:

```powershell
dcdiag /test:Advertising
```

**Result:** The Advertising test passed.

**Finding:** DC01 was successfully advertising itself as a Domain Controller.

### 5.6 SYSVOL and NETLOGON Verification

Command:

```powershell
Get-SmbShare
```

The following shares were found:

- SYSVOL
- NETLOGON

**Finding:** Both important Active Directory-related network shares were present.

### 5.7 Kerberos Key Distribution Center Verification

Command:

```powershell
Get-Service KDC
```

The KDC service was found.

The Key Distribution Center (KDC) is responsible for Kerberos authentication ticket issuance.

### 5.8 Kerberos Ticket Verification

Command:

```powershell
klist
```

The Kerberos ticket cache contained an entry for:

```text
krbtgt/CORP.TEST
```

This indicated that a Ticket Granting Ticket (TGT) was present in the current logon session.

The following command was then executed:

```powershell
klist get krbtgt
```

**Result:** The Kerberos ticket request completed successfully.

**Finding:** Kerberos ticket acquisition was working in the current logon session.

---

## 6. Verification Summary

The following checks were completed after the VirtualBox crash.

| Verification Item | Result |
|---|---|
| Windows Server Startup | PASS |
| NTDS Service | Running |
| Domain Information | Retrieved Successfully |
| DNS Service | Running |
| DNS Diagnostic | PASS |
| Domain Controller Advertising | PASS |
| SYSVOL Share | Present |
| NETLOGON Share | Present |
| KDC Service | Found |
| Kerberos TGT | Present |
| Kerberos Ticket Request | Successful |

### Overall Assessment

Based on the verification results, the Active Directory Domain Services installation and Domain Controller promotion appear to have completed successfully despite the VirtualBox crash.

The domain `corp.test` was accessible, the DNS diagnostic test passed, the Domain Controller Advertising test passed, and Kerberos ticket acquisition was successful.

However, these checks do not establish that every Active Directory component is fully healthy.

Further validation is recommended, including a complete `dcdiag` assessment and testing authentication from a separate Windows client machine.

---

## 7. Root Cause Analysis

The exact root cause of the VirtualBox crash has not yet been confirmed.

The investigation identified the following factors:

- An access violation occurred in `VBoxVMM.DLL`.
- A Windows hypervisor was detected on the host.
- Virtualization-Based Security was running.
- VirtualBox attempted to use the NEM fallback execution path.
- VirtualBox reported that AMD-V was not available for direct use.

These findings suggest that the host virtualization environment may have contributed to the issue.

However, further investigation is required before identifying a definitive root cause.

---

## 8. Lessons Learned

This troubleshooting exercise provided practical experience in:

1. Analysing VirtualBox crash logs.
2. Investigating Windows virtualization and hypervisor configurations.
3. Identifying access violation errors.
4. Verifying Active Directory Domain Services.
5. Performing DNS diagnostic tests.
6. Checking Domain Controller advertising.
7. Verifying SYSVOL and NETLOGON shares.
8. Inspecting Kerberos ticket caches.
9. Using PowerShell and Windows diagnostic tools to investigate infrastructure issues.

An important lesson was that a virtualization software crash does not necessarily mean that the guest operating system installation or configuration has failed.

The actual state of the guest system must be independently verified.

---

## 9. Next Steps

The following tasks are planned:

- [ ] Perform a complete Domain Controller diagnostic using `dcdiag`.
- [ ] Investigate the remaining DNS warning events.
- [ ] Verify the KDC service status is Running.
- [ ] Configure the Windows 11 CLIENT01 virtual machine.
- [ ] Join CLIENT01 to the `corp.test` domain.
- [ ] Verify domain user authentication.
- [ ] Capture Kerberos authentication traffic using Wireshark.
- [ ] Document the client-to-domain-controller authentication process.

---

## 10. Conclusion

The VirtualBox crash initially created uncertainty about the success of the Active Directory Domain Controller promotion.

Through systematic troubleshooting and post-crash verification, the Windows Server virtual machine was successfully restarted and the core Active Directory, DNS, and Kerberos functions were verified.

The investigation demonstrated the importance of collecting technical evidence, separating confirmed findings from hypotheses, and validating infrastructure services after an unexpected failure.
