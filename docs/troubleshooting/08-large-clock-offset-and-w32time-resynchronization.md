# Large Clock Offset and Windows Time Synchronization Recovery

## 1. Overview

This document records the investigation and recovery of a Windows Time synchronization issue in the Windows Server 2025 Active Directory lab.

DC01 was approximately nine days behind CLIENT01. Although an external NTP server was already configured, the Windows Time service was not successfully synchronizing.

The initial synchronization request was rejected because the required time correction was too large.

The issue was resolved by manually correcting the DC01 system clock, successfully synchronizing DC01 with the external NTP server, and then restoring domain-hierarchy synchronization on CLIENT01.

### Objectives

- Identify the cause of the large time difference.
- Understand the PDC Emulator's role in Active Directory time synchronization.
- Verify the external NTP configuration and connectivity.
- Investigate why Windows Time was not synchronizing.
- Correct the DC01 system clock.
- Restore external NTP synchronization on DC01.
- Restore domain-hierarchy synchronization on CLIENT01.
- Verify the final time sources and synchronization status.

---

## 2. Lab Environment

| Component | Configuration |
|---|---|
| Hypervisor | Oracle VirtualBox |
| Domain Controller | DC01 |
| Server OS | Windows Server 2025 |
| Domain | `corp.test` |
| Domain Controller FQDN | `dc01.corp.test` |
| DC01 Host-only IP | `192.168.56.10` |
| Domain Client | CLIENT01 |
| Client OS | Windows 11 |
| Client Host-only IP | `192.168.56.105` |
| External NTP server | `time.cloudflare.com` |
| Windows Time service | W32Time |

### 2.1 Time Zone Configuration

The DC01 time zone was previously verified as:

```text
Id          : Cen. Australia Standard Time
DisplayName : UTC+09:30 Adelaide
```

The Windows setting:

```text
Adjust for daylight saving time automatically
```

was enabled.

<img width="1597" height="1201" alt="image" src="https://github.com/user-attachments/assets/e6cb06eb-3fcb-4a10-a22b-1fb8490af090" />


**Important distinction:**

A time zone determines how UTC is displayed as local time. Daylight Saving Time adjusts the local time offset when applicable.

Neither setting performs NTP synchronization.

A correct time zone does not guarantee that the underlying system clock is synchronized.

---

## 3. Initial Problem

### 3.1 Observed Time Difference

The date displayed on DC01 was approximately nine days behind the date displayed on CLIENT01.

The client provided the following reference time during the recovery process:

```text
Friday, November 27, 2026 8:57:36 PM
```

DC01 was still displaying a date around November 18, 2026.

This large difference could affect Active Directory authentication.

### 3.2 Why Time Synchronization Matters

Active Directory uses Kerberos as an authentication protocol.

Kerberos relies on timestamps to help prevent replay attacks and to validate authentication requests.

The default maximum Kerberos clock skew in Windows domains is commonly five minutes.

A difference of approximately nine days is far beyond this tolerance and can cause Kerberos authentication failures.

### 3.3 Expected Time Synchronization Architecture

The expected time hierarchy was:

```text
          External NTP Server
          time.cloudflare.com
                   |
                   | NTP
                   v
             DC01.corp.test
             PDC Emulator
                   |
                   | Domain Time Hierarchy
                   v
                CLIENT01
```

DC01 is responsible for synchronizing with the external time source in this lab.

CLIENT01 should obtain its time through the Active Directory domain hierarchy.

---

## 4. Investigation

### 4.1 Confirm the PDC Emulator Role

**Execution location:** DC01

**GUI procedure:**

1. Open Server Manager.
2. Select Tools.
3. Open Active Directory Domains and Trusts.
4. Right-click the domain root.
5. Select Operations Master.

**Observed result:**

```text
dc01.corp.test
```

<img width="1597" height="1200" alt="image" src="https://github.com/user-attachments/assets/be6830db-3afd-40c6-b9da-2f6835484b20" />

<img width="541" height="997" alt="image" src="https://github.com/user-attachments/assets/117299ff-0896-487b-8a94-325c463787ee" />

<img width="1300" height="892" alt="image" src="https://github.com/user-attachments/assets/23073af2-b01f-4fce-ba7b-f4b2cf5eea69" />

<img width="568" height="417" alt="image" src="https://github.com/user-attachments/assets/d4c75555-6c77-4209-af48-27160ab8c8bc" />


DC01 was confirmed as the PDC Emulator.

**Purpose:**

The PDC Emulator is one of the five FSMO roles in Active Directory.

It has an important role in domain time synchronization. In the forest-root domain, the PDC Emulator normally obtains time from a reliable external time source.

Other domain controllers and domain-joined clients follow the Active Directory time hierarchy.

**Why was this checked?**

Before repairing time synchronization, it was necessary to identify which domain controller held the PDC Emulator role.

This identified DC01 as the appropriate server to investigate for external NTP synchronization.

### 4.2 Inspect Windows Time Configuration

**Execution location:** DC01

**Command:**

```powershell
w32tm /query /configuration
```

**Purpose:**

Displays the effective Windows Time service configuration.

This command was used to check whether an external NTP server was already configured.

**Observed values:**

| Setting | Observed value |
|---|---|
| NtpServer | `time.cloudflare.com,0x8 (local)` |
| Type | `NTP (local)` |
| AnnounceFlags | `5 (local)` |

**Interpretation:**

The external NTP server was already registered.

The `Type` value of `NTP` indicated that DC01 was configured to use manually specified NTP peers.

However, having an NTP server configured does not prove that the Windows Time service has successfully synchronized with it.

The configuration and actual synchronization state must be checked separately.

### 4.3 Check the Current Time Source

**Execution location:** DC01

**Command:**

```powershell
w32tm /query /source
```

**Purpose:**

Displays the time source currently selected by the Windows Time service.

**Observed result:**

```text
Free-running System Clock
```

<img width="655" height="124" alt="image" src="https://github.com/user-attachments/assets/66d7400a-ecb5-4dd5-948f-500dd897669a" />


**Interpretation:**

DC01 was not currently using the configured external NTP server as its synchronized time source.

It was operating using its own system clock.

This indicated that the NTP configuration existed, but successful synchronization had not been established.

### 4.4 Check the NTP Peer State

**Execution location:** DC01

**Command:**

```powershell
w32tm /query /peers
```

**Purpose:**

Displays the configured time peers and their current states.

**Observed result:**

```text
State: Active
```

The `Last Successful Sync Time` field was not displayed in the output examined.

**Interpretation:**

The peer was recognized as active by Windows Time.

However, `Active` does not necessarily mean that synchronization has succeeded.

Peer state and successful synchronization are different pieces of information.

### 4.5 Check Windows Time Service Status

**Execution location:** DC01

**Command:**

```powershell
Get-Service W32Time
```

**Purpose:**

Checks whether the Windows Time service is running.

**Observed result:**

```text
Status: Running
```

**Interpretation:**

The Windows Time service was running.

The issue was therefore not simply that the service had stopped.

### 4.6 Measure the Difference Between DC01 and the External NTP Server

**Execution location:** DC01

**Command:**

```powershell
w32tm /stripchart /computer:time.cloudflare.com /samples:3 /dataonly
```

**Purpose:**

Measures the time offset between the local computer and the specified external NTP server.

**Observed result:**

```text
14:02:28 +801716.2358248s
14:02:30 +801716.2354310s
14:02:32 +801716.2368327s
```

**Interpretation:**

The reported offset was approximately:

```text
801716 seconds
```

This is approximately 9.28 days.

The measurement confirmed that a very large time difference existed between DC01 and the external NTP reference.

**Important distinction:**

`w32tm /stripchart` measures the time difference.

It does not change the system clock or perform synchronization.

It is a diagnostic command, not a repair command.

### 4.7 Check the Windows Time Synchronization Status

**Execution location:** DC01

**Command:**

```powershell
w32tm /query /status
```

**Purpose:**

Displays the current synchronization status, including:

- Leap Indicator
- Stratum
- Last Successful Sync Time
- Source
- Poll Interval

**Observed initial result:**

```text
Leap Indicator: 3 (not synchronized)
```

The selected source was also reported as:

```text
Free-running System Clock
```

**Interpretation:**

Windows Time did not consider DC01 synchronized.

The `Leap Indicator: 3` value indicated that the time service was not synchronized.

---

## 5. Initial Synchronization Attempt

### 5.1 Request Time Synchronization

**Execution location:** DC01

**Command:**

```powershell
w32tm /resync /rediscover
```

**Purpose:**

Requests synchronization with the configured time source.

**Parameter explanation:**

| Parameter | Purpose |
|---|---|
| `w32tm` | Windows Time service command-line utility |
| `/resync` | Requests a time synchronization operation |
| `/rediscover` | Rediscovers available time sources before synchronization |

**Observed error:**

```text
Sending resync command to local computer
The computer did not resync because the required time change was too big.
```

### 5.2 Error Analysis

The error indicated that Windows Time rejected the requested correction because the time difference was too large.

At this point, the following facts had been established:

| Investigation item | Result |
|---|---|
| External NTP server configured | Yes |
| NTP peer state | Active |
| W32Time service | Running |
| External NTP measurement | Successful |
| Current source | Free-running System Clock |
| Leap Indicator | 3 |
| Synchronization request | Rejected |
| Error reason | Required time change was too big |

**Finding:**

The immediate obstacle was the large time correction required by Windows Time.

The evidence did not indicate that the NTP hostname was missing or that the W32Time service was stopped.

The next step was to bring DC01's system clock close to the correct time before requesting synchronization again.

---

## 6. Resolution: Correct the DC01 System Clock

### 6.1 Obtain the Correct Reference Time

**Reference machine:** CLIENT01

The client displayed:

```text
Friday, November 27, 2026 8:57:36 PM
```

This time was used as the reference for correcting DC01.

### 6.2 Manually Correct the DC01 Clock

**Execution location:** DC01

**PowerShell command:**

```powershell
Set-Date -Date "11/27/2026 20:57:36"
```

<img width="876" height="288" alt="image" src="https://github.com/user-attachments/assets/9c563948-bc6f-46f6-a82f-49a153909268" />


**Purpose:**

Manually sets the local system date and time.

**Parameter explanation:**

| Element | Purpose |
|---|---|
| `Set-Date` | Changes the local computer's system date and time |
| `-Date` | Specifies the date and time to set |
| `11/27/2026` | November 27, 2026 |
| `20:57:36` | 8:57:36 PM in 24-hour format |

**Observed result:**

The DC01 system clock changed successfully.

**Why was this necessary?**

The previous synchronization request was rejected because the required correction was too large.

Manually bringing the clock close to the reference time reduced the correction required from the external NTP source.

**Important distinction:**

`Set-Date` changes the local computer's clock.

It does not synchronize the computer with an NTP server.

The actual NTP synchronization still needed to be completed afterward.

---

## 7. Restore External NTP Synchronization on DC01

### 7.1 Request Synchronization Again

**Execution location:** DC01

**Command:**

```powershell
w32tm /resync /rediscover
```

**Purpose:**

Requests synchronization after the system clock has been corrected.

**Observed result:**

The synchronization request completed successfully.

<img width="730" height="180" alt="image" src="https://github.com/user-attachments/assets/faef1cf9-be60-4451-8369-07e2644b15c1" />


### 7.2 Verify the Selected Time Source

**Execution location:** DC01

**Command:**

```powershell
w32tm /query /source
```

**Observed result:**

```text
time.cloudflare.com
```

<img width="667" height="124" alt="image" src="https://github.com/user-attachments/assets/60d66fd0-aca9-4750-8273-b90b971dc8e3" />


**Interpretation:**

DC01 was now using the external NTP server as its selected time source.

This was an important confirmation that the time source had changed from the free-running system clock to the configured external NTP server.

### 7.3 Verify the Synchronization Status

**Execution location:** DC01

**Command:**

```powershell
w32tm /query /status
```

**Observed result:**

| Field | Final observation |
|---|---|
| Leap Indicator | `0 (no warning)` |
| Last Successful Sync Time | Updated |
| Source | `time.cloudflare.com` |

**Interpretation:**

`Leap Indicator: 0` indicates that Windows Time no longer reports the unsynchronized condition.

The updated successful synchronization time and external source confirmed that DC01's external NTP synchronization had recovered.

<img width="745" height="375" alt="image" src="https://github.com/user-attachments/assets/a315b672-efe5-4e6b-bca4-d3cdd87f4750" />


---

## 8. Restore Domain-Hierarchy Synchronization on CLIENT01

After DC01 had successfully synchronized with the external NTP server, the client was checked.

### 8.1 Check the Client's Initial Time Source

**Execution location:** CLIENT01

**Command:**

```powershell
w32tm /query /source
```

**Observed result:**

```text
Local CMOS Clock
```

<img width="634" height="66" alt="image" src="https://github.com/user-attachments/assets/852d981d-1197-4b68-b6e9-3f200ad61075" />


**Interpretation:**

CLIENT01 was using its local clock rather than obtaining time from the domain hierarchy.

The client needed to be configured to use the Active Directory domain time hierarchy.

### 8.2 Configure Domain-Hierarchy Synchronization

**Execution location:** CLIENT01

**Command:**

```powershell
w32tm /config /syncfromflags:domhier /update
```

**Purpose:**

Configures the Windows Time service to obtain time through the Active Directory domain hierarchy.

**Parameter explanation:**

| Parameter | Purpose |
|---|---|
| `w32tm` | Windows Time service command-line utility |
| `/config` | Changes Windows Time configuration |
| `/syncfromflags:domhier` | Uses the Active Directory domain hierarchy as the time source selection method |
| `/update` | Notifies the Windows Time service that its configuration has changed |

**Observed result:**

```text
The command completed successfully.
```

<img width="909" height="91" alt="image" src="https://github.com/user-attachments/assets/4213d7e6-c605-4473-9c38-79ed03ddb653" />


**Interpretation:**

CLIENT01 was configured to follow the domain time hierarchy rather than relying on a manually specified external NTP peer.

### 8.3 Request Client Synchronization

**Execution location:** CLIENT01

**Command:**

```powershell
w32tm /resync /rediscover
```

**Purpose:**

Requests the client to rediscover its domain time source and synchronize.

**Observed result:**

The synchronization request completed successfully.

<img width="706" height="117" alt="image" src="https://github.com/user-attachments/assets/930be192-b1e3-4120-b184-11c9cf5719d6" />


### 8.4 Verify the Client's Selected Time Source

**Execution location:** CLIENT01

**Command:**

```powershell
w32tm /query /source
```

**Observed result:**

```text
DC01.corp.test
```

<img width="637" height="91" alt="image" src="https://github.com/user-attachments/assets/2c16d73f-39c0-4763-93a9-9475e366e7ac" />


**Interpretation:**

CLIENT01 was now using DC01 as its time source.

This matched the expected Active Directory domain time hierarchy.

---

## 9. Complete Command Reference

This section summarizes all commands used during this troubleshooting session.

| Command | Execution location | Purpose |
|---|---|---|
| `w32tm /query /configuration` | DC01 | Inspect effective Windows Time configuration |
| `w32tm /query /source` | DC01 / CLIENT01 | Identify the currently selected time source |
| `w32tm /query /peers` | DC01 | Inspect configured NTP peers and their states |
| `Get-Service W32Time` | DC01 | Check whether the Windows Time service is running |
| `w32tm /stripchart /computer:time.cloudflare.com /samples:3 /dataonly` | DC01 | Measure the offset against the external NTP server |
| `w32tm /query /status` | DC01 | Inspect synchronization status |
| `w32tm /resync /rediscover` | DC01 / CLIENT01 | Request synchronization and rediscover time sources |
| `Set-Date -Date "11/27/2026 20:57:36"` | DC01 | Manually correct the local system clock |
| `w32tm /config /syncfromflags:domhier /update` | CLIENT01 | Configure synchronization through the AD domain hierarchy |

### 9.1 Important Command Differences

**`w32tm /query /source`**

Displays the currently selected time source.

It does not perform synchronization.

**`w32tm /query /configuration`**

Displays the configured Windows Time settings.

A configured NTP server does not automatically mean synchronization has succeeded.

**`w32tm /query /peers`**

Displays peer information.

An `Active` peer state alone does not prove that the last synchronization was successful.

**`w32tm /stripchart`**

Measures the time difference between the local computer and a specified remote time source.

It does not correct the local clock.

**`w32tm /resync /rediscover`**

Requests actual synchronization.

The result must be checked afterward using the source and status commands.

**`Set-Date`**

Changes the local system clock manually.

It is not a replacement for configuring and verifying NTP synchronization.

**`w32tm /config /syncfromflags:domhier /update`**

Configures a domain-joined client to follow the Active Directory time hierarchy.

It does not configure an external NTP peer.

---

## 10. Questions and Technical Explanations

The following section records the technical questions raised during the troubleshooting process.

### Q1. Why did we need to check the PDC Emulator?

The PDC Emulator has a special role in the Active Directory time hierarchy.

The PDC Emulator in the forest-root domain normally obtains time from a reliable external source.

Other domain controllers and domain-joined computers follow the domain hierarchy.

Checking this role identified which server should be investigated as the external time synchronization point.

### Q2. What is a PDC Emulator?

PDC Emulator is one of the five Flexible Single Master Operations (FSMO) roles in Active Directory.

It is not a separate server product.

It is a special role assigned to a Domain Controller.

Among its responsibilities, it has an important role in domain time synchronization and certain password and authentication-related operations.

In this lab, `dc01.corp.test` was confirmed as the PDC Emulator.

### Q3. Why was the NTP server configured even though synchronization was not working?

There is an important difference between configuration and operation.

For example:

```text
NtpServer: time.cloudflare.com,0x8
Type: NTP
```

shows that an external peer is configured.

However, this does not prove that Windows Time has successfully synchronized with that peer.

The selected source and synchronization status must also be checked.

### Q4. What does `Free-running System Clock` mean?

It means Windows Time is not currently synchronized with a recognized external or domain time source.

The computer continues using its own system clock.

This is different from successfully synchronizing with an NTP server.

### Q5. What does `State: Active` mean in the peer output?

It means the peer is recognized as active by the Windows Time service.

It does not necessarily mean that the computer has successfully synchronized with that peer.

A successful synchronization must be verified separately.

### Q6. What does `Local Computer` mean in the resynchronization message?

When Windows displays:

```text
Sending resync command to local computer
```

`Local Computer` means the computer on which the command was executed.

For example:

- If the command is executed on DC01, the local computer is DC01.
- If the command is executed on CLIENT01, the local computer is CLIENT01.

It does not mean that the synchronization source is necessarily the local hardware clock.

The selected source must be checked separately using:

```powershell
w32tm /query /source
```

### Q7. Why did the synchronization request fail with "the required time change was too big"?

The initial time difference was approximately nine days.

Windows Time rejected the requested correction because the required time change exceeded the permitted correction range.

The error was:

```text
The computer did not resync because the required time change was too big.
```

The system clock was therefore manually corrected before requesting synchronization again.

### Q8. Why did we use `Set-Date`?

`Set-Date` allowed DC01's system clock to be moved close to the correct reference time.

This reduced the large difference that had caused the previous synchronization request to fail.

Afterward, NTP synchronization was requested again.

### Q9. Does `w32tm /stripchart` synchronize the time?

No.

It measures the offset between the local computer and a remote time source.

It is useful for diagnosis, but it does not change the system clock.

The actual synchronization request is performed using:

```powershell
w32tm /resync /rediscover
```

### Q10. Why was CLIENT01 using `Local CMOS Clock`?

The client was not using the expected domain time source at that point.

It was using its local clock.

The client was configured to use the Active Directory domain hierarchy with:

```powershell
w32tm /config /syncfromflags:domhier /update
```

After requesting synchronization, the selected source changed to:

```text
dc01.corp.test
```

### Q11. What is the difference between the DC's external NTP configuration and the client's domain hierarchy configuration?

They are two different synchronization roles.

| Computer | Synchronization method | Expected source |
|---|---|---|
| DC01 | External NTP | `time.cloudflare.com` |
| CLIENT01 | Active Directory domain hierarchy | `dc01.corp.test` |

The client should not independently use an external NTP peer when it is intended to follow the domain hierarchy.

### Q12. Why was the daylight saving time setting enabled?

The setting:

```text
Adjust for daylight saving time automatically
```

was enabled.

This allows Windows to adjust the local time offset when daylight saving rules apply.

It is separate from NTP synchronization.

The daylight saving setting was not identified as the cause of the nine-day clock difference.

### Q13. Why was the Internet Time tab missing?

The Windows Server 2025 interface did not display the expected Internet Time tab in the Date and Time control panel view used during this investigation.

The Windows Settings interface also differed from the expected Windows 11 instructions.

Rather than continuing to search for an unavailable GUI option, the Windows Time service was diagnosed and repaired using PowerShell.

The operating system being investigated was Windows Server 2025. CLIENT01 was the Windows 11 client.

### Q14. Why is it important to check both the source and the status?

The source indicates where Windows currently obtains time.

The status indicates whether Windows considers itself synchronized and when the last successful synchronization occurred.

For example:

```text
Source: time.cloudflare.com
Leap Indicator: 0 (no warning)
```

Together with an updated successful synchronization time, these provide stronger evidence of successful synchronization than the source alone.

---

### 11.1 Final Time Synchronization Architecture

```text
          time.cloudflare.com
             External NTP
                  |
                  | Successful NTP synchronization
                  v
             DC01.corp.test
              PDC Emulator
                  |
                  | Active Directory
                  | Domain Hierarchy
                  v
             CLIENT01
```

**Overall result:**

DC01 successfully recovered external NTP synchronization.

CLIENT01 was configured to follow the domain time hierarchy and successfully completed its synchronization request.

The final selected source on the client was confirmed as `DC01.corp.test`.

---

## 12. Key Technical Learnings

### 12.1 Configuration Does Not Equal Successful Synchronization

A configured NTP peer does not guarantee that the Windows Time service is synchronized.

Always verify:

1. The configured peer.
2. The selected source.
3. The peer state.
4. The synchronization status.
5. The last successful synchronization time.

### 12.2 The PDC Emulator Has an Important Role in Time Management

The PDC Emulator in the forest-root domain normally obtains time from an external reliable source.

Domain clients and other domain controllers follow the Active Directory time hierarchy.

### 12.3 Large Clock Differences Can Prevent Synchronization

Windows Time can reject a synchronization request if the required correction is too large.

The error message must be interpreted before repeatedly requesting synchronization.

In this case, manually correcting the system clock allowed the subsequent synchronization request to succeed.

### 12.4 Time Zone and NTP Are Different

A time zone controls local time display.

NTP synchronizes the underlying system clock with a time source.

Changing a time zone cannot replace NTP synchronization.

### 12.5 Kerberos Depends on Accurate Time

Kerberos authentication uses timestamps.

The default maximum clock skew is commonly five minutes.

Accurate synchronization is therefore important for reliable Active Directory authentication.

---

## 13. Conclusion

The time synchronization issue was caused by a large difference between the DC01 system clock and the external NTP reference.

Although `time.cloudflare.com` was already configured, DC01 was using its free-running system clock and reported that it was not synchronized.

An initial synchronization request was rejected because the required time correction was too large.

The system clock was manually corrected using the time displayed on CLIENT01. After that correction, DC01 successfully synchronized with `time.cloudflare.com`.

CLIENT01 initially used `Local CMOS Clock`. Its Windows Time configuration was changed to follow the Active Directory domain hierarchy, and synchronization completed successfully.

The final selected sources were:

- DC01: `time.cloudflare.com`
- CLIENT01: `dc01.corp.test`

This troubleshooting exercise demonstrated the importance of understanding the Active Directory time hierarchy, diagnosing Windows Time service behavior, distinguishing configuration from successful synchronization, and verifying recovery using multiple pieces of evidence.
