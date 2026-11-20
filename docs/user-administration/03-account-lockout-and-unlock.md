# Account Lockout and Unlocking

## 1. Overview

This lab demonstrates how to configure an Active Directory account lockout policy, trigger an account lockout, verify the locked account state, and unlock the account.

The exercise simulates a common Help Desk support scenario:

> A user cannot sign in because their Active Directory account has been locked after repeated authentication failures.

### Objectives

- Understand the Active Directory Account Lockout Policy.
- Configure the domain account lockout settings.
- Create and use a dedicated test account.
- Verify an account lockout using PowerShell.
- Unlock an account using Active Directory Users and Computers (ADUC).
- Confirm that the user can sign in after unlocking.

### Lab Environment

| Component | Configuration |
|---|---|
| Domain Controller | DC01 |
| Client | CLIENT01-NEW |
| Operating System | Windows Server 2025 / Windows 11 |
| Domain | corp.test |
| Test Account | testuser01 |
| Account Management Tool | Active Directory Users and Computers |
| PowerShell | Windows PowerShell |

---

## 2. Create a Dedicated Test Account

A dedicated test account was created to avoid affecting existing accounts.

**Account Information**

| Attribute | Value |
|---|---|
| First Name | Test |
| Last Name | User |
| Full Name | Test User 01 |
| SamAccountName | testuser01 |
| Description | Help Desk Lab Test Account |
| Department | IT |
| Job Title | Lab Test Account |
| Company | Lab Environment |
| Office | Adelaide |

**Execution Location:** DC01

**Tool:** Active Directory Users and Computers (ADUC)

The account was created under:

`corp.test → Users`

The account was initially configured to require a password change at the next logon.

### PowerShell Verification

```powershell
Get-ADUser -Identity testuser01
```


<img width="1015" height="594" alt="image" src="https://github.com/user-attachments/assets/58154e46-494f-45c6-ac93-4a591800068e" />



This command retrieves the AD user object and verifies that the account exists.

Additional attributes were checked using:

```powershell
Get-ADUser -Identity testuser01 -Properties Description,Department,Title,Company,Office |
Select-Object Name,SamAccountName,Description,Department,Title,Company,Office
```

**Purpose:**

- `-Properties` retrieves additional AD attributes.
- `Select-Object` displays only the selected attributes.

The account information was successfully verified.



<img width="1605" height="987" alt="image" src="https://github.com/user-attachments/assets/6c175903-7558-416c-baf1-083817cfc87f" />



---

## 3. Inspect the Existing Account Lockout Policy

**Execution Location:** DC01

**Tool:** Windows PowerShell

Before changing the policy, the existing domain settings were inspected.

```powershell
Get-ADDefaultDomainPasswordPolicy |
Format-List LockoutThreshold,LockoutDuration,LockoutObservationWindow
```


<img width="1585" height="282" alt="image" src="https://github.com/user-attachments/assets/46237154-2294-4e90-b614-590c19d6700a" />



### Initial Configuration

| Attribute | Initial Value | Meaning |
|---|---|---|
| LockoutThreshold | 0 | Account lockout disabled |
| LockoutDuration | 00:10:00 | Lockout duration if enabled |
| LockoutObservationWindow | 00:10:00 | Observation period for failed attempts |

### Understanding the Attributes

**LockoutThreshold**

Specifies the number of failed sign-in attempts before an account is locked.

A value of `0` means account lockout is disabled by this domain policy.

**LockoutDuration**

Specifies how long an account remains locked before it is automatically unlocked.

**LockoutObservationWindow**

Specifies the time period during which failed authentication attempts are counted.

---

## 4. Configure the Account Lockout Policy

**Execution Location:** DC01

**Tool:** Windows PowerShell (Administrator)

The domain account lockout policy was configured for the lab.

```powershell
Set-ADDefaultDomainPasswordPolicy -Identity "corp.test" -LockoutThreshold 5 -LockoutDuration (New-TimeSpan -Minutes 10) -LockoutObservationWindow (New-TimeSpan -Minutes 10)
```

### Command Explanation

| Parameter | Value | Purpose |
|---|---|---|
| `-Identity` | corp.test | Specifies the target domain |
| `-LockoutThreshold` | 5 | Locks the account after 5 failed attempts |
| `-LockoutDuration` | 10 minutes | Specifies the lockout duration |
| `-LockoutObservationWindow` | 10 minutes | Defines the observation period |

`New-TimeSpan -Minutes 10` creates a time interval of 10 minutes.


<img width="1594" height="129" alt="image" src="https://github.com/user-attachments/assets/e7fcb0f1-c8a5-4dfe-b96e-31a76137a52a" />



### Verify the New Configuration

```powershell
Get-ADDefaultDomainPasswordPolicy -Identity "corp.test" |
Format-List LockoutThreshold,LockoutDuration,LockoutObservationWindow
```

Expected configuration:

```text
LockoutThreshold         : 5
LockoutDuration          : 00:10:00
LockoutObservationWindow : 00:10:00
```


<img width="1591" height="223" alt="image" src="https://github.com/user-attachments/assets/116cd9b2-7bad-42c5-9522-9f8b05606ce5" />



### Understanding the Configuration

The policy is configured to lock an account when the configured threshold of five failed authentication attempts is reached within the observation window.

The lockout duration is ten minutes.

**Important:** This is a domain-level policy and can affect other domain accounts, not only the test account.

---

## 5. Sign In Using the Test Account

**Execution Location:** CLIENT01-NEW

The dedicated test account was used to perform the authentication test.

Username:

```text
CORP\testuser01
```

During the first sign-in, Windows requested a password change because the account had been configured with the option:

`User must change password at next logon`

The initial password was entered, and a new lab password was configured.

After the password change, the account successfully signed in.

The current account was checked using:

```powershell
whoami
```

Expected result:

```text
corp\testuser01
```

This confirmed that the client was using the intended domain account.


<img width="688" height="294" alt="image" src="https://github.com/user-attachments/assets/a4d46fa6-2c20-4163-9ac1-681b69008fa8" />


---

## 6. Trigger and Verify Account Lockout

An intentional authentication failure was performed using the dedicated test account.

**Execution Location:** CLIENT01-NEW

**Test Account:** CORP\testuser01

An incorrect password was entered during the sign-in process.

The following authentication error was displayed:

```text
Invalid credentials, delaying next attempt
```

This indicated that the authentication attempt had failed.


<img width="570" height="726" alt="image" src="https://github.com/user-attachments/assets/162eee28-49d7-4de6-8d96-4c97abf24089" />



<img width="571" height="663" alt="image" src="https://github.com/user-attachments/assets/c44bb94b-8e45-4608-a110-9afe5ad15940" />



<img width="769" height="633" alt="image" src="https://github.com/user-attachments/assets/c1daea26-2b92-447a-96e2-8ae39db67ca4" />



<img width="736" height="640" alt="image" src="https://github.com/user-attachments/assets/04b66714-d1a3-4520-92ab-387b344fef9e" />




### Verify the Account State

**Execution Location:** DC01

**Tool:** Windows PowerShell

The account state was inspected using:

```powershell
Get-ADUser -Identity testuser01 -Properties BadPwdCount,LockedOut |
Select-Object SamAccountName,BadPwdCount,LockedOut
```

### Attribute Explanation

| Attribute | Meaning |
|---|---|
| SamAccountName | The account's logon name |
| BadPwdCount | Recorded count of incorrect password attempts |
| LockedOut | Indicates whether the account is currently locked |

### Observed Result

The `LockedOut` attribute returned:

```text
True
```

This confirmed that Active Directory had placed the test account into a locked state.


<img width="1587" height="237" alt="image" src="https://github.com/user-attachments/assets/a53dbf7a-31a3-45fc-b5e0-72c665f3cfd8" />



**Note:** The configured threshold was five attempts. The actual `BadPwdCount` value should be recorded from the captured PowerShell output rather than assuming the number of attempts from the lockout status alone.

---

## 7. Unlock the Account Using ADUC

The account was unlocked through the graphical interface rather than PowerShell.

**Execution Location:** DC01

**Tool:** Active Directory Users and Computers (ADUC)

### Procedure

1. Open Server Manager.
2. Select `Tools`.
3. Open `Active Directory Users and Computers`.
4. Navigate to `corp.test → Users`.
5. Locate `testuser01`.
6. Right-click the account and select `Properties`.
7. Open the `Account` tab.
8. Select the `Unlock account` option.
9. Click `Apply`, then `OK`.

This operation removed the account lockout state.

### Why Use ADUC?

Help Desk technicians frequently use ADUC to perform routine Active Directory account administration.

The graphical interface provides a straightforward way to inspect and manage user account properties.


<img width="1123" height="795" alt="image" src="https://github.com/user-attachments/assets/d8a163a7-515c-43cd-a935-dd4d233a325f" />


<img width="612" height="810" alt="image" src="https://github.com/user-attachments/assets/6bc53136-fee7-48d7-8c89-75ccfa867f1f" />


<img width="616" height="807" alt="image" src="https://github.com/user-attachments/assets/4ee800f3-eba4-486b-aff6-e1b5c1b1498a" />



---

## 8. Verify the Account Was Unlocked

**Execution Location:** DC01

**Tool:** Windows PowerShell

After unlocking the account through ADUC, the account state was checked again.

```powershell
Get-ADUser -Identity testuser01 -Properties LockedOut |
Select-Object SamAccountName,LockedOut
```

### Expected Result

```text
SamAccountName  LockedOut
--------------  ---------
testuser01      False
```

The observed result was:

```text
LockedOut : False
```

This confirmed that the account had been successfully unlocked.


<img width="1588" height="199" alt="image" src="https://github.com/user-attachments/assets/f407639d-d95d-4741-b96a-57eab3d3bd43" />



---

## 9. Verify Successful Sign-In After Unlocking

**Execution Location:** CLIENT01-NEW

The test account was used to sign in again.

Username:

```text
CORP\testuser01
```

The password configured during the initial sign-in was entered.

The account successfully signed in after the unlock operation.

This confirmed that the account lockout had been cleared and normal authentication was working again.


<img width="595" height="718" alt="image" src="https://github.com/user-attachments/assets/6814c052-0ea9-4726-8a3e-a6e254c125aa" />


<img width="504" height="678" alt="image" src="https://github.com/user-attachments/assets/9f96b2d8-90c5-4f5b-aae3-2ea783c36340" />



<img width="1594" height="1210" alt="image" src="https://github.com/user-attachments/assets/d857d232-36ef-475f-9a15-34cf07273fb1" />




---

## 10. Key Technical Concepts

| Concept | Description |
|---|---|
| Account Lockout | Temporarily prevents an account from authenticating after reaching the configured failed-attempt threshold |
| LockoutThreshold | Number of failed authentication attempts required to trigger a lockout |
| LockoutDuration | How long an account remains locked |
| LockoutObservationWindow | Period used to count failed authentication attempts |
| BadPwdCount | Attribute that records incorrect password attempts |
| LockedOut | Indicates the account's current lockout state |
| ADUC | Graphical tool for managing Active Directory objects |
| Unlock-ADAccount | PowerShell cmdlet used to unlock an AD account |

### Important PowerShell Commands

| Command | Purpose |
|---|---|
| `Get-ADDefaultDomainPasswordPolicy` | Retrieves the domain password policy |
| `Set-ADDefaultDomainPasswordPolicy` | Modifies the domain password policy |
| `Get-ADUser` | Retrieves Active Directory user information |
| `Unlock-ADAccount` | Unlocks an Active Directory account |

---

## 11. Lab Completion Summary

The following tasks were completed:

- [x] Created a dedicated AD test account.
- [x] Verified user account information using PowerShell.
- [x] Inspected the existing domain account lockout policy.
- [x] Configured the account lockout threshold and timing.
- [x] Verified the updated policy.
- [x] Signed in using the dedicated test account.
- [x] Triggered an account lockout.
- [x] Verified `LockedOut : True`.
- [x] Unlocked the account using ADUC.
- [x] Verified `LockedOut : False`.
- [x] Confirmed successful sign-in after unlocking.

**Result:** Successfully demonstrated the Active Directory account lockout and unlock workflow using both GUI and PowerShell.

