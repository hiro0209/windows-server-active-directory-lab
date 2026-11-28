# Login and Authentication Troubleshooting

## 1. Objective

Investigate a Windows domain login failure in an Active Directory environment.

The objectives were to:

- Investigate a domain login failure.
- Examine Windows Security Event ID 4625.
- Interpret the authentication failure status code.
- Check the Active Directory user account status.
- Investigate the relationship between the authentication failure and the previously identified time synchronisation issue.
- Verify successful domain login after restoring time synchronisation.

## 2. Lab Environment

| Component | Configuration |
|---|---|
| Domain Controller | DC01 |
| Operating System | Windows Server 2025 |
| Domain Name | corp.test |
| DC IP Address | 192.168.56.10 |
| Client Computer | CLIENT01-NEW |
| Client OS | Windows 11 |
| Client IP Address | 192.168.56.105 |
| Test User | testuser01 |

## 3. Investigating the Login Failure

### 3.1 Investigating the Authentication Issue

A domain login failure was investigated using the Windows client and Active Directory environment.

The investigation focused on identifying the failed logon events recorded by Windows Security auditing.

**Execution location:** DC01

### 3.2 Checking the Current User Context

The following command was used earlier in the investigation.

```powershell
whoami /all
```

**Purpose:**

Identify the currently logged-in user and examine their security context.

**Command explanation:**

- `whoami`: Displays the current logged-in account.
- `/all`: Displays the user's SID, group memberships, and privileges.

This was used to examine the current domain account and its security permissions.

<img width="1590" height="1048" alt="image" src="https://github.com/user-attachments/assets/973b933d-989b-4e22-954f-3cd66f36d64e" />


<img width="1587" height="1068" alt="image" src="https://github.com/user-attachments/assets/a5189fab-4751-4114-aefd-7a2075da8726" />


### 3.3 Checking Failed Logon Events (Event ID 4625)

The following command was executed on DC01.

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 10 | Select-Object TimeCreated, Id, Message | Format-List
```

**Purpose:**

Retrieve the 10 most recent failed logon events from the Windows Security event log and display their timestamps, event IDs, and detailed messages.

**Command explanation:**

| Command / Parameter | Meaning |
|---|---|
| `Get-WinEvent` | Retrieves Windows event log records. |
| `-FilterHashtable` | Filters events using specified conditions. |
| `LogName='Security'` | Searches the Security event log. |
| `Id=4625` | Selects failed logon events (Event ID 4625). |
| `-MaxEvents 10` | Retrieves up to 10 matching events. |
| `Select-Object` | Selects specific properties to display. |
| `TimeCreated` | Shows when the event occurred. |
| `Id` | Shows the event ID. |
| `Message` | Shows the detailed event message. |
| `Format-List` | Displays the selected properties in list format. |

**Important Event ID:**

Event ID 4625 represents a failed logon attempt.

The event details were examined to identify the account involved, the time of the failure, and the authentication failure status.

**Screenshot — Failed Logon Event**

<img width="862" height="1039" alt="image" src="https://github.com/user-attachments/assets/286ce801-da60-43cc-8efa-7b2d6bbf4671" />

### 3.4 Analysing the Authentication Failure Details

The Event ID 4625 record contained the following information.

**Subject**

| Field | Value |
|---|---|
| Security ID | `S-1-0-0` |
| Account Name | `-` |
| Account Domain | `-` |
| Logon ID | `0x0` |

**Logon Type**

The Logon Type field was examined in the event details.

**Account For Which Logon Failed**

| Field | Value |
|---|---|
| Security ID | `S-1-0-0` |
| Account Name | `-` |
| Account Domain | `-` |

**Failure Information**

| Field | Value |
|---|---|
| Failure Reason | An error occurred during logon. |
| Status | `0xC000006D` |
| SubStatus | `0xC0000133` |

**Process Information**

| Field | Value |
|---|---|
| Caller Process ID | `0x0` |
| Caller Process Name | `-` |

**Network Information**

| Field | Value |
|---|---|
| Workstation Name | `-` |
| Source Network Address | `-` |
| Source Port | `-` |

**Detailed Authentication Information**

| Field | Value |
|---|---|
| Logon Process | Kerberos |
| Authentication Package | Kerberos |
| Transited Services | `-` |
| Package Name (NTLM only) | `-` |
| Key Length | `0` |

**Analysis of Status Codes**

- `0xC000006D` (`STATUS_LOGON_FAILURE`): Indicates that the logon attempt failed.
- `0xC0000133` (`STATUS_TIME_DIFFERENCE_AT_DC`): Indicates a time difference associated with the Domain Controller authentication process.

The SubStatus provides more specific information about the failure than the general Status code.

The recorded authentication package was Kerberos, indicating that Kerberos was involved in the failed authentication attempt.

**Important Note:**

The event did not identify the account name, workstation name, or source network address. Therefore, the specific user and client associated with this particular event could not be conclusively identified from these fields alone.

### 3.5 Checking the Active Directory Account

The Active Directory account was checked to determine whether the failure was related to the user's account state.

**Execution location:** DC01

```powershell
Get-ADUser testuser01
```

**Purpose:**

Retrieve the Active Directory user object for `testuser01`.

**Command explanation:**

- `Get-ADUser`: Retrieves information about an Active Directory user.
- `testuser01`: Specifies the user account to retrieve.

The account information examined included:

| Attribute | Observed result |
|---|---|
| Name | Test User |
| SamAccountName | testuser01 |
| Enabled | True |
| LockedOut | False |
| PasswordExpired | False |
| PasswordLastSet | 11/17/2026 9:02:10 PM |

**Analysis:**

- The account was enabled.
- The account was not locked out.
- The password had not expired.

Therefore, these account conditions did not explain the authentication failure.


<img width="1597" height="367" alt="image" src="https://github.com/user-attachments/assets/3ac0be2a-0af2-48eb-9233-2016f62484c8" />


## 4. Authentication Investigation Findings

The Event ID 4625 record contained the following authentication information:

| Field | Value |
|---|---|
| Logon Process | Kerberos |
| Authentication Package | Kerberos |
| Status | `0xC000006D` |
| SubStatus | `0xC0000133` |

The recorded SubStatus was associated with a time difference involving the Domain Controller.

A significant time synchronisation issue had previously been identified and resolved in a separate Windows Time Service troubleshooting exercise.

However, the available event information did not identify the failed account or workstation. Therefore, the specific relationship between this event and the `testuser01` login attempt could not be conclusively established.

## 5. Active Directory Account Verification

The Active Directory account `testuser01` was checked using PowerShell.

The account was confirmed to be:

- Enabled.
- Not locked out.
- Not password-expired.

These results indicated that the checked account properties did not reveal an account-state issue.

## 6. Login Verification

A subsequent domain login test was performed using `testuser01`.

**Result:** Successful domain login.

This confirmed that the account could successfully log in to the domain during the subsequent test.

## 7. Conclusion

This investigation demonstrated how to examine Windows Security Event ID 4625 and review Kerberos-related authentication failure information.

The investigation included:

- Retrieving failed logon events.
- Examining Status and SubStatus values.
- Reviewing Kerberos authentication information.
- Checking Active Directory account properties.
- Performing a subsequent domain login test.

A previous time synchronisation issue had been resolved separately. Although the recorded SubStatus was associated with a time difference involving the Domain Controller, the available evidence did not conclusively establish that this was the cause of the specific `testuser01` login failure.
