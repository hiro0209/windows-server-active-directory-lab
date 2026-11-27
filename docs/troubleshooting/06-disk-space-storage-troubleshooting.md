# Disk Space and Storage Troubleshooting

## 1. Objective

The objective of this lab was to investigate disk space usage and storage health on a Windows 11 client (`CLIENT01`).

The investigation focused on:

- Checking available disk space.
- Identifying and removing unnecessary temporary files.
- Investigating user profile storage.
- Checking virtual disk health.
- Analyzing Windows disk-related events.
- Investigating NTFS health.
- Verifying filesystem integrity using CHKDSK.

The investigation was performed using Windows Settings, File Explorer, and PowerShell.

---

## 2. Environment

| Component | Details |
|---|---|
| Client | Windows 11 |
| Computer Name | CLIENT01 |
| Virtualization | Oracle VirtualBox |
| Disk | Virtual Disk |
| Disk Size | 80 GB |
| Filesystem | NTFS |
| Investigation Account | `corp\Administrator` |

---

## 3. Disk Space Investigation

### 3.1 Check Disk Space Using File Explorer

Opened File Explorer and navigated to:

`This PC`

Checked the available storage space on the Windows (C:) drive.

**Purpose:**

To quickly identify whether the system drive has insufficient available storage.

Low disk space can affect Windows updates, application installation, temporary file creation, and overall system operation.

<img width="985" height="742" alt="image" src="https://github.com/user-attachments/assets/4a984082-fa27-4ced-ae0b-614688ec4f09" />


### 3.2 Check Volume Information Using PowerShell

**Command:**

```powershell
Get-Volume -DriveLetter C
```

**Purpose:**

To retrieve information about the C: volume, including its filesystem, capacity, drive letter, and operational status.

**Command explanation:**

| Parameter | Meaning |
|---|---|
| `Get-Volume` | Retrieves information about Windows storage volumes. |
| `-DriveLetter C` | Specifies the C: drive. |

This provides a more detailed view of the volume than File Explorer.

<img width="1213" height="336" alt="image" src="https://github.com/user-attachments/assets/89a5ac28-7488-4850-9b2a-796cc24a1d37" />


### 3.3 Investigate Storage Usage Through Windows Settings

Navigated to:

`Settings → System → Storage`

Reviewed the storage categories to identify areas consuming disk space.

**Purpose:**

To identify which categories of data are using storage and determine whether unnecessary files can be safely removed.


<img width="1278" height="1000" alt="image" src="https://github.com/user-attachments/assets/83f9559c-d80b-49a5-8ebb-54bb92ce12cb" />


### 3.4 Remove Temporary Files

Navigated to:

`Settings → System → Storage → Temporary files`

The following categories were identified:

| Category | Size |
|---|---:|
| Delivery Optimization Files | 209 MB |
| Temporary files | 129 MB |
| Thumbnails | 7.04 MB |
| Temporary Internet Files | 2.91 MB |
| DirectX Shader Cache | 28 KB |
| **Total displayed** | **Approximately 348 MB** |

The selected temporary files were removed using Windows Settings.

**Purpose:**

To reclaim disk space by removing unnecessary temporary data.

Temporary files and caches may accumulate over time. Removing appropriate temporary files can free storage without requiring the deletion of personal documents or installed applications.

**Observed result:**

| Measurement | Available Space |
|---|---:|
| Before cleanup | 43.91 GB |
| After cleanup | 44.11 GB |
| Increase | Approximately 0.20 GB |

The displayed categories totalled approximately 348 MB, while the measured increase in available space was approximately 200 MB.

The difference was observed but its exact cause was not determined.

<img width="1275" height="1146" alt="image" src="https://github.com/user-attachments/assets/067abf7b-5819-4a43-a035-099ebd8ddcf6" />

<img width="798" height="1006" alt="image" src="https://github.com/user-attachments/assets/0a2c5524-9b02-41e6-93f2-8aa4fed58b7c" />

<img width="1273" height="1138" alt="image" src="https://github.com/user-attachments/assets/9fb6b7aa-d2d8-4c58-bcd3-3522d3b2f17d" />

<img width="1233" height="352" alt="image" src="https://github.com/user-attachments/assets/1e685e5a-1ddd-450e-a8cd-a218b4cbe878" />


---

## 4. User Profile Storage Investigation

### 4.1 Identify User Profile Directories

**Command:**

```powershell
Get-ChildItem C:\Users -Directory
```

**Purpose:**

To list directories under `C:\Users` and identify Windows user profiles.

**Command explanation:**

| Parameter | Meaning |
|---|---|
| `Get-ChildItem` | Retrieves files and directories from a specified location. |
| `C:\Users` | Specifies the location of Windows user profile directories. |
| `-Directory` | Returns directories rather than files. |

User profiles may contain large amounts of data, including:

- Desktop files
- Downloads
- Documents
- AppData
- Browser caches
- Application data

These locations can contribute significantly to disk usage.

<img width="799" height="397" alt="image" src="https://github.com/user-attachments/assets/4297ca0a-4cab-481e-ba0b-173dee29b476" />


### 4.2 Investigate Access Permissions

During the investigation, the `testuser01` account was unable to access the `hiro` user profile directory.

**Command:**

```powershell
whoami
```

**Purpose:**

To identify the currently authenticated Windows account.

**Observed output:**

```text
corp\testuser01
```

The account was subsequently changed to:

```text
corp\Administrator
```

The profile directory could then be accessed.

**Technical explanation:**

Windows uses access control lists (ACLs) to control access to files and directories.

A user may be unable to inspect another user's profile because of access restrictions.

This is an expected security behavior and does not necessarily indicate a filesystem problem.

---

## 5. Virtual Disk Health Investigation

### 5.1 Check Physical Disk Information

**Command:**

```powershell
Get-PhysicalDisk
```

**Purpose:**

To retrieve information about storage devices recognized by Windows, including operational status and health status.

**Command explanation:**

| Property | Meaning |
|---|---|
| `FriendlyName` | Device name presented by Windows. |
| `MediaType` | Media type, such as SSD or HDD, when identified. |
| `OperationalStatus` | Operational state of the device. |
| `HealthStatus` | Reported health condition. |
| `Size` | Device capacity. |
| `CanPool` | Indicates whether the disk can be added to a Storage Spaces pool. |

**Observed result:**

| Property | Result |
|---|---|
| FriendlyName | VBOX HARDDISK |
| MediaType | Unspecified |
| OperationalStatus | OK |
| HealthStatus | Healthy |
| Size | 80 GB |
| CanPool | False |

The storage device was identified as a VirtualBox virtual disk.

<img width="1105" height="216" alt="image" src="https://github.com/user-attachments/assets/f92ff576-477c-491e-8624-972ad04127e2" />


**Interpretation:**

Windows reported the virtual disk as operational and healthy.

However, this result does not independently verify the physical health of the host computer's storage device.

---

## 6. Windows Disk Event Investigation

### 6.1 Search for Disk Error Events

**Command:**

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Disk'; Level=2} -MaxEvents 10
```

**Purpose:**

To investigate Error-level events generated by the Disk provider in the Windows System event log.

**Command explanation:**

| Parameter | Meaning |
|---|---|
| `Get-WinEvent` | Retrieves Windows event log records. |
| `-FilterHashtable` | Specifies filtering criteria. |
| `LogName='System'` | Searches the System event log. |
| `ProviderName='Disk'` | Filters events generated by the Disk provider. |
| `Level=2` | Filters Error-level events. |
| `-MaxEvents 10` | Returns a maximum of 10 matching events. |

**Observed result:**

```text
No events were found that match the specified selection criteria.
```

No matching Disk-provider Error-level events were found using this query.

<img width="641" height="103" alt="image" src="https://github.com/user-attachments/assets/43458528-e1da-4a7d-bb41-1fea2745cd48" />


### 6.2 Understand Windows Event Severity Levels

| Level | Name | Meaning |
|---|---|---|
| 1 | Critical | Severe failure requiring immediate attention. |
| 2 | Error | An error occurred. |
| 3 | Warning | A potential issue or condition requiring attention. |
| 4 | Information | Normal informational event. |
| 5 | Verbose | Detailed diagnostic information. |

A Warning does not automatically mean a device has failed.

The severity level must be interpreted alongside the event message, frequency, and other diagnostic evidence.

### 6.3 Search Disk Events Across All Levels

**Command:**

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Disk'; Level=1,2,3,4,5} -MaxEvents 20
```

**Purpose:**

To inspect Disk-provider events across all five severity levels rather than restricting the investigation to Error events.

**Observed result:**

A Warning event with Event ID 153 was identified.


<img width="638" height="116" alt="image" src="https://github.com/user-attachments/assets/f9ca4d76-72bd-48c1-b4a8-3ef0a5ba3b69" />


---

## 7. Investigate Event ID 153

### 7.1 Retrieve the Full Event Message

**Command:**

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Disk'; Id=153} -MaxEvents 5 | Format-List TimeCreated, Id, LevelDisplayName, Message
```

**Purpose:**

To retrieve the full message of Event ID 153 and examine the details of the disk-related warning.

**Command explanation:**

| Parameter | Meaning |
|---|---|
| `Id=153` | Filters events by Event ID 153. |
| `-MaxEvents 5` | Retrieves up to five matching events. |
| `Format-List` | Displays selected event properties in a list format. |
| `TimeCreated` | Event occurrence time. |
| `LevelDisplayName` | Severity level. |
| `Message` | Detailed event message. |

**Observed event:**

| Property | Value |
|---|---|
| TimeCreated | 11/19/2026 8:13:23 PM |
| Event ID | 153 |
| Level | Warning |
| Disk | Disk 0 |

**Message:**

```text
The IO operation at logical block address 0x336ce90
for Disk 0 (PDO name: \Device\0000001f) was retried.
```

<img width="788" height="101" alt="image" src="https://github.com/user-attachments/assets/6f277ab1-7102-4b2b-aa24-dc48e25adce7" />


**Interpretation:**

The message indicates that a disk I/O operation was retried.

A retry means that the original I/O operation did not complete successfully on its initial attempt, and Windows retried the operation.

Possible causes can include temporary storage latency or issues somewhere along the storage I/O path.

The event itself does not establish the root cause or prove that the disk is failing.

### 7.2 Check Event ID 153 Frequency

**Command:**

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Disk'; Id=153; StartTime=(Get-Date).AddDays(-30)} -ErrorAction SilentlyContinue | Measure-Object
```

**Purpose:**

To count Event ID 153 occurrences during the previous 30 days.

**Command explanation:**

| Parameter | Meaning |
|---|---|
| `StartTime` | Specifies the beginning of the search period. |
| `Get-Date` | Retrieves the current date and time. |
| `.AddDays(-30)` | Subtracts 30 days from the current date. |
| `-ErrorAction SilentlyContinue` | Suppresses non-terminating errors. |
| `Measure-Object` | Counts or calculates statistics on pipeline objects. |

**Observed result:**

One Event ID 153 occurrence was found during the previous 30 days.

<img width="798" height="133" alt="image" src="https://github.com/user-attachments/assets/93e93e3b-c7e4-45ba-bcab-68bae92db470" />


**Interpretation:**

The event was isolated within the searched period.

No evidence of repeated Event ID 153 occurrences was found during that 30-day check.

However, an isolated event does not identify its cause.

---

## 8. Investigate Other Storage-Related Providers

### 8.1 Search Related Storage Events

**Command:**

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; StartTime=(Get-Date).AddDays(-30)} -MaxEvents 300 | Where-Object {$_.ProviderName -match 'disk|stor|ntfs|volsnap'} | Select-Object TimeCreated, ProviderName, Id, LevelDisplayName
```

**Purpose:**

To examine recent System events associated with storage-related providers.

**Command explanation:**

| Command or parameter | Meaning |
|---|---|
| `Get-WinEvent` | Retrieves Windows event log records. |
| `LogName='System'` | Searches the System event log. |
| `StartTime` | Sets the beginning of the time range. |
| `-MaxEvents 300` | Retrieves up to 300 events. |
| `Where-Object` | Filters objects based on a condition. |
| `$_.ProviderName` | Accesses the ProviderName property of the current event. |
| `-match` | Checks whether a value matches a regular expression. |
| `Select-Object` | Selects specific properties for display. |

**Providers included in the search pattern:**

| Provider | General purpose |
|---|---|
| Disk | Disk I/O-related events. |
| Stor | Storage subsystem-related events. |
| NTFS | NTFS filesystem events. |
| Volsnap | Volume Shadow Copy-related events. |

**Important limitation:**

The command retrieves only the latest 300 System events before filtering them.

Therefore, it does not guarantee that all storage-related events from the previous 30 days were examined.

**Observed result:**

The following events were identified:

| Provider | Event ID | Level |
|---|---|---|
| Microsoft-Windows-Ntfs | 98 | Information |

Multiple Event ID 98 records were observed.

<img width="1216" height="529" alt="image" src="https://github.com/user-attachments/assets/e4aeeaed-6ca6-4622-b7c4-427ea8995c17" />


---

## 9. Investigate NTFS Event ID 98

### 9.1 Retrieve Full NTFS Event Messages

**Command:**

```powershell
Get-WinEvent -FilterHashtable @{LogName='System'; ProviderName='Microsoft-Windows-Ntfs'; Id=98} -MaxEvents 3 | Format-List TimeCreated, Id, LevelDisplayName, Message
```

**Purpose:**

To retrieve the full message of NTFS Event ID 98 and determine the reported filesystem condition.

**Observed results:**

Three volumes were reported as healthy.

**Example:**

```text
Volume C: (\Device\HarddiskVolume4) is healthy.
No action is needed.
```

Other volumes also reported:

```text
is healthy. No action is needed.
```

<img width="1219" height="429" alt="image" src="https://github.com/user-attachments/assets/7453a09a-f2d2-40c1-b7bf-783f24a20c9b" />


**Interpretation:**

NTFS reported the examined volumes as healthy at the time of these events.

Event ID 98 was Information-level, not Warning or Error.

This is positive evidence regarding the reported NTFS volume condition, but it does not prove that every possible filesystem or storage problem is absent.

---

## 10. Filesystem Integrity Verification Using CHKDSK

### 10.1 Run CHKDSK in Read-Only Mode

**Command:**

```powershell
chkdsk C:
```

**Purpose:**

To examine the C: drive filesystem for logical filesystem errors without performing repairs.

**Command explanation:**

| Parameter | Meaning |
|---|---|
| `chkdsk` | Checks a disk or volume for filesystem errors. |
| `C:` | Specifies the target volume. |
| No `/F` parameter | Performs the check without requesting filesystem repairs. |

The following warning was displayed:

```text
WARNING! /F parameter not specified.
Running CHKDSK in read-only mode.
```

This confirms that CHKDSK was running in read-only mode.


<img width="1594" height="1120" alt="image" src="https://github.com/user-attachments/assets/0dbe48c8-6e5d-4263-9f3f-96151fef0815" />


### 10.2 CHKDSK Stages

CHKDSK examined several aspects of the filesystem.

| Stage | Purpose |
|---|---|
| Stage 1 | Examines basic filesystem structure and file records. |
| Stage 2 | Examines filename linkage and directory indexes. |
| Stage 3 | Examines security descriptors and related filesystem metadata. |
| USN Journal verification | Checks the Update Sequence Number Journal. |

**Observed results:**

```text
0 bad file records processed.
```

```text
0 unindexed files scanned.
```

```text
Security descriptor verification completed.
```

```text
USN Journal verification completed.
```

**Final result:**

```text
Windows has scanned the file system and found no problems.

No further action is required.
```

**Interpretation:**

CHKDSK completed its filesystem examination and reported no filesystem problems.

No repair operation was performed because the command was executed without the `/F` parameter.

The output did not include a bad-sector summary, so a zero bad-sector result cannot be claimed based on this execution.

---

## 11. Final Findings

| Investigation | Result |
|---|---|
| Initial available storage | 43.91 GB |
| Available storage after cleanup | 44.11 GB |
| Temporary file cleanup | Completed |
| Disk device status | Healthy |
| Disk Error-level events | None found by the query |
| Event ID 153 | One occurrence within 30 days |
| NTFS Event ID 98 | Volumes reported healthy |
| CHKDSK | No filesystem problems found |
| Filesystem repairs | Not performed |

### Overall Assessment

The investigation identified temporary files that could be removed, resulting in an increase in available disk space.

The virtual disk was reported as Healthy by Windows.

An isolated Disk Event ID 153 Warning was identified, indicating that a disk I/O operation had been retried. However, only one occurrence was found during the 30-day frequency check.

NTFS Event ID 98 reported the examined volumes as healthy, and CHKDSK completed its read-only filesystem examination without detecting problems.

Based on the evidence collected, no ongoing storage or filesystem failure was identified during this investigation.

The root cause of the isolated Event ID 153 was not determined.

---

## 12. Troubleshooting Workflow

```text
User reports low disk space or storage issue
                 |
                 v
      Check available disk space
                 |
                 v
       Investigate Storage usage
                 |
                 v
       Remove safe temporary files
                 |
                 v
      Check user profile directories
                 |
                 v
      Check physical disk status
                 |
                 v
      Review Disk-related events
                 |
                 v
      Investigate Event ID 153
                 |
                 v
      Check event frequency
                 |
                 v
      Investigate NTFS events
                 |
                 v
       Run CHKDSK read-only
                 |
                 v
      Evaluate all collected evidence
```

---

## 13. Commands Reference

| Command | Purpose |
|---|---|
| `Get-Volume -DriveLetter C` | Check volume information. |
| `Get-ChildItem C:\Users -Directory` | List user profile directories. |
| `whoami` | Identify the currently authenticated user. |
| `Get-PhysicalDisk` | Check storage device status and reported health. |
| `Get-WinEvent` | Retrieve Windows event logs. |
| `-FilterHashtable` | Filter events by properties such as log, provider, ID, and time. |
| `-MaxEvents` | Limit the number of returned events. |
| `Format-List` | Display selected properties in list format. |
| `Measure-Object` | Count or calculate statistics on pipeline objects. |
| `Get-Date` | Retrieve the current date and time. |
| `.AddDays(-30)` | Calculate a date 30 days before the current date. |
| `Where-Object` | Filter pipeline objects based on a condition. |
| `Select-Object` | Select specific properties for display. |
| `chkdsk C:` | Check the C: filesystem in read-only mode. |

---

## 14. Skills Demonstrated

- Windows disk space investigation.
- Temporary storage cleanup.
- User profile directory investigation.
- Windows filesystem and volume administration.
- PowerShell event log filtering.
- Windows Event severity interpretation.
- Disk I/O warning analysis.
- NTFS health investigation.
- CHKDSK read-only verification.
- Evidence-based troubleshooting.
- Distinguishing observed facts from potential causes.
