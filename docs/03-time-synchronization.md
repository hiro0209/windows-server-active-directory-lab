# Windows Server Time Synchronization Configuration and Troubleshooting

## 1. Overview

This document records the time synchronization work completed in the Windows Server 2025 Active Directory lab.

The goal was to make DC01 obtain time from an external NTP source and then confirm that CLIENT01-NEW followed the Active Directory domain time hierarchy.

This is the implementation record: it focuses on the actual lab topology, commands, configuration changes, troubleshooting decisions, and observed results. For the protocol concepts behind NTP, see [NTP Fundamentals](https://github.com/hiro0209/network-engineering-notes/blob/main/03-application/ntp/01-ntp-fundamentals.md).

## 2. Objectives

- Identify the initial time source used by DC01.
- Provide DC01 with a network path to an external NTP server.
- Configure the Windows Time service on DC01.
- Confirm that CLIENT01-NEW uses the domain time hierarchy.
- Investigate the client's large reported clock offset.
- Measure the final time difference between CLIENT01-NEW and DC01.

## 3. Lab Environment

| Component | Configuration |
|---|---|
| Hypervisor | Oracle VirtualBox 7.2.8 |
| Domain Controller | DC01 |
| Server OS | Windows Server 2025 |
| Domain | `corp.test` |
| NetBIOS domain name | `CORP` |
| DC01 Host-only address | `192.168.56.10` |
| DC01 NAT address | `10.0.3.15` |
| Domain Client | CLIENT01-NEW |
| Client OS | Windows 11 Enterprise Evaluation |
| Client address | `192.168.56.105` |
| Client DNS server | `192.168.56.10` |
| Client time zone | `Australia/Adelaide` |
| External NTP peer configured on DC01 | `time.cloudflare.com` |

### 3.1 Network Design

DC01 uses two virtual network adapters:

| Adapter | VirtualBox mode | Purpose |
|---|---|---|
| Adapter 1 | Host-only | Private communication between DC01 and CLIENT01-NEW |
| Adapter 2 | NAT | Outbound connectivity from DC01 to the external NTP service |

CLIENT01-NEW remains connected to the Host-only network. The NAT adapter was added to DC01 without replacing its existing Host-only adapter.

## 4. Initial Diagnosis on DC01

### 4.1 Inspect the Network Configuration

**VM:** DC01

**Command:**

```powershell
ipconfig
```

**Purpose:** Display the IP configuration of the network adapters, including IPv4 addresses and the default gateway.

**Why was it used?** The external NTP server could not initially be reached. The network path had to be checked before changing the time-service configuration.

**Observed finding:** The Host-only adapter did not have a default gateway. A Host-only network provides private connectivity within the lab; it does not automatically provide an external route.

### 4.2 Identify the Current Time Source

**VM:** DC01

**Command:**

```powershell
w32tm /query /source
```

**Purpose:** Display the time source currently selected by the Windows Time service.

**Why was it used?** This establishes whether Windows is using an external time peer, a domain source, or the local hardware clock.

**Observed result:**

```text
Local CMOS Clock
```

**Interpretation:** DC01 was using its local hardware clock rather than a confirmed external NTP source.

### 4.3 Inspect the Windows Time Status

**VM:** DC01

**Command:**

```powershell
w32tm /query /status
```

**Purpose:** Display synchronization status, including the source, stratum, last successful synchronization, and timing metadata.

**Why was it used?** The selected source alone does not show whether the service has synchronized successfully or how it currently assesses the time source.

**Result note:** The command was run, but the complete output was not preserved in the available session record. Exact DC01 stratum and last-sync values are therefore not claimed here.

### 4.4 Test the External NTP Peer

**VM:** DC01

**Command:**

```powershell
w32tm /stripchart /computer:time.cloudflare.com /samples:5 /dataonly
```

**Purpose:** Request repeated time measurements from `time.cloudflare.com` and display the estimated offset.

**Why was it used?** This checks the time source using Windows Time functionality and is more relevant to NTP than a generic TCP port test.

**Initial result:** The command did not successfully return measurements before the NAT adapter was added.

**Interpretation:** This matched the network finding that the Host-only adapter had no external route.

## 5. Provide External Connectivity to DC01

### 5.1 Add a NAT Adapter

**VM:** DC01 (VirtualBox configuration; DC01 was shut down before changing the adapter settings)

**Procedure:**

1. Shut down DC01.
2. Open the VirtualBox settings for DC01.
3. Keep Adapter 1 configured as Host-only.
4. Enable Adapter 2 and set it to NAT.
5. Start DC01.

**Purpose:** Give DC01 outbound connectivity to the external NTP service while preserving the private Host-only connection used by the domain client.

**Observed result:** Windows detected the NAT adapter as `Ethernet2`, and it received:

```text
10.0.3.15
```

### 5.2 Resolve the External NTP Hostname

**VM:** DC01

**Command:**

```powershell
Resolve-DnsName time.cloudflare.com
```

**Purpose:** Resolve the NTP hostname through DNS.

**Why was it used?** The configured peer is a hostname. DNS resolution needed to be checked separately from NTP communication.

**Observed result:** The hostname resolved successfully after the NAT adapter was added.

**Interpretation:** This confirmed DNS resolution, but DNS success alone does not confirm that NTP packets can reach the server.

### 5.3 Understand the TCP Port 123 Test

**VM:** DC01

**Command:**

```powershell
Test-NetConnection time.cloudflare.com -Port 123
```

**Purpose:** Test whether a TCP connection can be established to the specified host and port.

**Observed result:** The TCP connection test failed.

**Troubleshooting explanation:** NTP normally uses **UDP port 123**, whereas `Test-NetConnection -Port 123` tests TCP by default. The failed TCP test did not establish that NTP/UDP was blocked. It was not a valid direct test of the NTP service.

### 5.4 Re-test the NTP Peer

**VM:** DC01

**Command:**

```powershell
w32tm /stripchart /computer:time.cloudflare.com /samples:5 /dataonly
```

**Purpose:** Measure the time offset to the external NTP peer.

**Observed result:** The command returned an offset measurement of approximately:

```text
+01.2793661
```

**Interpretation:** DC01 could now obtain a time measurement from the external peer. This was the relevant successful NTP test after adding the NAT adapter.

## 6. Configure the Windows Time Service on DC01

### 6.1 Configure the Manual Peer

**VM:** DC01

**Command:**

```powershell
w32tm /config /manualpeerlist:"time.cloudflare.com,0x8" /syncfromflags:manual /reliable:yes /update
```

**Purpose:** Configure `time.cloudflare.com` as a manual peer and update the Windows Time service configuration.

**Command breakdown:**

| Parameter | Purpose |
|---|---|
| `w32tm` | Windows Time service command-line utility |
| `/config` | Changes Windows Time configuration |
| `/manualpeerlist:"time.cloudflare.com,0x8"` | Sets the manual peer and its peer flags |
| `time.cloudflare.com` | External NTP hostname |
| `0x8` | Configures the peer association for client mode |
| `/syncfromflags:manual` | Uses the manually configured peer list as the source-selection mode |
| `/reliable:yes` | Marks this computer as a reliable time source in the domain hierarchy |
| `/update` | Notifies the service that the configuration has changed |

**Observed result:** The command returned a successful completion message.

**Important role check:** `/reliable:yes` is intended for the appropriate authoritative domain time source, normally the PDC Emulator in the forest-root domain. The PDC Emulator role should be verified before treating DC01 as the correct authoritative source. The configuration command succeeding does not prove that DC01 holds that role.

### 6.2 Request Synchronization

**VM:** DC01

**Command:**

```powershell
w32tm /resync /rediscover
```

**Purpose:** Request immediate synchronization and rediscovery of available time sources.

**Command breakdown:**

| Parameter | Purpose |
|---|---|
| `/resync` | Requests a synchronization operation |
| `/rediscover` | Rediscovers time sources before synchronization |

**Observed result:** The command returned a successful completion message.

### 6.3 Verify the DC01 Source and Status

**VM:** DC01

**Commands:**

```powershell
w32tm /query /source
w32tm /query /status
```

**Purpose:** Confirm which time source is selected and inspect the synchronization status.

**Result note:** These commands were run after the synchronization request, but the complete output was not preserved. The exact final source, stratum, and last successful synchronization time remain unverified in this document.

### 6.4 Verify the PDC Emulator Role

**VM:** DC01

**Command to run for final verification:**

```powershell
Get-ADDomain | Select-Object PDCEmulator
```

**Purpose:** Display the domain controller holding the PDC Emulator FSMO role for the domain.

**Why is it important?** The PDC Emulator in the forest-root domain normally acts as the authoritative point for the AD time hierarchy. This check is needed before confirming that DC01 is the correct domain time authority.

**Result note:** The command was requested during troubleshooting, but its output was not captured in the available session record.

## 7. Diagnose and Synchronize CLIENT01-NEW

### 7.1 Check the Client Time Configuration

**VM:** CLIENT01-NEW

**Command:**

```powershell
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\W32Time\Parameters" | Select-Object Type, NtpServer
```

**Purpose:** Read the Windows Time service's configured time type and NTP server value from the registry.

**Observed result:**

```text
Type      : NT5DS
NtpServer : time.windows.com,0x9
```

**Interpretation:** `NT5DS` means that the client follows the Active Directory domain time hierarchy. The presence of `time.windows.com,0x9` in the `NtpServer` registry value does not by itself mean that the client is using that peer directly while its type is `NT5DS`.

### 7.2 Check the Client's Selected Time Source

**VM:** CLIENT01-NEW

**Command:**

```powershell
w32tm /query /source
```

**Purpose:** Display the time source currently selected by the Windows Time service.

**Observed result:**

```text
dc01.corp.test
```

**Interpretation:** The client selected DC01 as its time source, which is consistent with the expected domain time hierarchy.

### 7.3 Review Windows Time Events

**VM:** CLIENT01-NEW

**Command:**

```powershell
Get-WinEvent -FilterHashtable @{ LogName = 'System'; ProviderName = 'Microsoft-Windows-Time-Service' } -MaxEvents 10 | Format-List TimeCreated, Id, Message
```

**Purpose:** Retrieve recent Windows Time service events from the System event log.

**Why was it used?** Event messages can provide evidence of large offsets and synchronization problems that are not fully explained by the current source alone.

**Command breakdown:**

| Element | Purpose |
|---|---|
| `Get-WinEvent` | Retrieves event log records |
| `-FilterHashtable` | Applies structured filters |
| `LogName = 'System'` | Searches the System event log |
| `ProviderName = 'Microsoft-Windows-Time-Service'` | Filters for Windows Time events |
| `-MaxEvents 10` | Returns up to 10 matching events |
| `Format-List` | Displays selected event properties in a list |
| `TimeCreated, Id, Message` | Shows event time, event ID, and event details |

**Observed findings:**

- Event ID 52 reported a clock offset of approximately 60,824 seconds.
- Event ID 50 reported a difference greater than 5,000 milliseconds for 900 seconds.

60,824 seconds is approximately 16 hours, 53 minutes, and 44 seconds.

**Interpretation:** The event log confirmed that the client had experienced a substantial time difference. The event messages do not, by themselves, prove that the time-zone setting was the root cause. A time-zone display difference and a real underlying clock offset must be distinguished.

### 7.4 Review the Client Time Configuration and Status

**VM:** CLIENT01-NEW

**Commands used during diagnosis:**

```powershell
w32tm /query /configuration
w32tm /query /status
```

**Purpose:** Inspect the effective Windows Time configuration and the current synchronization status.

**Recorded status details from the earlier client check:**

| Field | Recorded value |
|---|---|
| Stratum | `0 (unspecified)` |
| Precision | `-23` |
| Root delay | `0.0047412s` |
| Root dispersion | `18.1303482s` |
| Reference ID | `0x00000000` |
| Last successful synchronization | `11/16/2026 8:53:45 pm` |

A later check reportedly showed the stratum changing from 0 to 2 without a manually issued resynchronization command. The complete later output was not preserved, so the exact transition cannot be independently verified from the saved record.

### 7.5 Request Client Synchronization

**VM:** CLIENT01-NEW

**Command:**

```powershell
w32tm /resync /rediscover
```

**Purpose:** Ask the client to rediscover the domain time source and request synchronization.

**Observed result:** The command returned a successful completion message.

## 8. Time Zone Configuration

During troubleshooting, DC01 was changed from `Pacific Standard Time` to the Adelaide-region Windows time-zone ID.

**VM:** DC01

**Command used:**

```powershell
Set-TimeZone -Id "Cen. Australia Standard Time"
```

**Verification command:**

```powershell
Get-TimeZone
```

**Observed result:** The time zone was shown as `Cen. Australia Standard Time`.

**Technical distinction:** A time-zone setting controls the conversion between UTC and the local displayed time. It does not synchronize the system clock with an NTP peer. The actual clock offset must be checked separately.

## 9. Final Offset Measurement

### 9.1 Measure CLIENT01-NEW Against DC01

**VM:** CLIENT01-NEW

**Command:**

```powershell
w32tm /stripchart /computer:dc01.corp.test /samples:5 /dataonly
```

**Purpose:** Measure the clock offset between the domain client and DC01.

**Observed measurements:**

```text
-01.8208272s
-02s
-03s
-03s
```

**Interpretation:** The measured difference was approximately 1–3 seconds at the time of testing. This is substantially below the typical five-minute Kerberos clock-skew tolerance. It is evidence that the clocks were closely aligned during this measurement, but it does not replace checking the time source and status on both systems.

## 10. Troubleshooting Summary

| Symptom | Command / action | Finding or resolution |
|---|---|---|
| DC01 used its local hardware clock | `w32tm /query /source` | Source was `Local CMOS Clock`. |
| External NTP test initially failed | `w32tm /stripchart ...` and `ipconfig` | Host-only adapter had no default gateway; an external route was missing. |
| DC01 needed an external path | Added VirtualBox Adapter 2 as NAT | `Ethernet2` received `10.0.3.15`. |
| External hostname needed checking | `Resolve-DnsName time.cloudflare.com` | DNS resolution succeeded. |
| TCP port 123 test failed | `Test-NetConnection ... -Port 123` | The command tested TCP, not NTP/UDP. |
| External time measurement succeeded | `w32tm /stripchart ...` | Returned approximately `+01.2793661`. |
| Client reported a large time difference | `Get-WinEvent` for `Microsoft-Windows-Time-Service` | Event ID 52 reported approximately 60,824 seconds. |
| Client needed to update its time | `w32tm /resync /rediscover` | Synchronization request completed successfully. |
| Final client/DC offset needed checking | `w32tm /stripchart /computer:dc01.corp.test ...` | Measured approximately 1–3 seconds. |

## 11. Final Verification and Outstanding Checks

| Verification | Result |
|---|---|
| DC01 Host-only address | `192.168.56.10` |
| DC01 NAT address | `10.0.3.15` |
| External NTP hostname resolution | Successful |
| External NTP measurement after NAT was added | Successful; approximately `+01.2793661` |
| DC01 manual peer configuration command | Completed successfully |
| DC01 resynchronization request | Completed successfully |
| Client time configuration type | `NT5DS` |
| Client selected source | `dc01.corp.test` |
| Client resynchronization request | Completed successfully |
| Final measured client/DC offset | Approximately 1–3 seconds |
| DC01 final source/status output | Full output not preserved |
| Client final status output | Full output not preserved |
| PDC Emulator role on DC01 | Not yet verified from captured output |

### 11.1 Recommended Final Checks

Run these commands and save their complete output before marking the configuration fully verified.

**VM: DC01**

```powershell
Get-ADDomain | Select-Object PDCEmulator
w32tm /query /source
w32tm /query /status
```

**VM: CLIENT01-NEW**

```powershell
w32tm /query /source
w32tm /query /status
```

These outputs establish the PDC Emulator role, selected time sources, stratum, and last successful synchronization times.

## 12. Outcome and Next Stage

DC01 initially used `Local CMOS Clock` and did not have an external route through its Host-only adapter. After a NAT adapter was added, DC01 could resolve and measure the external NTP peer. The Windows Time service was configured with `time.cloudflare.com`, and the client was observed using `dc01.corp.test` as its time source with `NT5DS` configured.

The final measurement between CLIENT01-NEW and DC01 was approximately 1–3 seconds. This provides a useful baseline before analyzing Kerberos authentication traffic in Wireshark.

The remaining verification items are the captured PDC Emulator role and the complete final `w32tm /query /status` output from both machines.
