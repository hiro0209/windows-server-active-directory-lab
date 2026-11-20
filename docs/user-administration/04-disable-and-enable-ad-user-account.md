# Active Directory User Account Disable and Enable

## 1. Overview

### Objective

This lab demonstrates how to disable and re-enable an Active Directory domain user account using the Active Directory Users and Computers (ADUC) graphical interface.

After changing the account status, the `net user` command is used to verify whether the account is disabled or enabled.

### Environment

| Item | Configuration |
|---|---|
| Server OS | Windows Server 2025 |
| Domain Controller | DC01 |
| Client Computer | CLIENT01-NEW |
| Domain Name | corp.test |
| Test Account | testuser01 |
| Management Tool | Active Directory Users and Computers (ADUC) |
| Verification Tool | Windows PowerShell |

### Scenario

A user account needs to be temporarily disabled by an administrator.

For example, an employee may leave an organisation or require their account access to be suspended.

The administrator must disable the account, verify its status, and re-enable it when access is required again.

| Attribute | Value |
|---|---|
| Username | testuser01 |
| Full Name | Test User 01 |
| Description | Help Desk Lab Test Account |
| Department | IT |
| Domain | corp.test |

---

## 2. Disabling a User Account

### Step 1: Locate the Test User Account

**Execution Location:** DC01

Open Active Directory Users and Computers (ADUC).

Navigate to:

`corp.test` → `Users`

Locate the test account:

`testuser01`

**Purpose:**

Before changing an account's status, the administrator must identify the correct user account.

Verifying the target account helps prevent accidental changes to another user's access.

**Screenshot 01: User Account Before Disabling**

The screenshot shows the test account before the disable operation.



<img width="1129" height="799" alt="image" src="https://github.com/user-attachments/assets/a90b5cab-f63f-491f-85b4-14beac7aa38d" />



### Step 2: Disable the User Account

**Execution Location:** DC01

In Active Directory Users and Computers:

1. Right-click `testuser01`.
2. Select `Disable Account`.
3. Confirm the operation if prompted.

**Purpose:**

Disabling an account prevents the user from authenticating using that domain account.

This is commonly used when an organisation needs to suspend a user's access.

Unlike an account lockout, an administratively disabled account does not automatically become enabled after a specified period.

**Screenshot 02: User Account Disabled**

The screenshots show the disabled account in ADUC. The account icon displays a downward arrow, indicating that the account is disabled.


<img width="286" height="495" alt="image" src="https://github.com/user-attachments/assets/6839990f-5b0f-4749-9515-adab86591c0a" />









<img width="417" height="258" alt="image" src="https://github.com/user-attachments/assets/c79e8504-373e-45c9-a0ae-5f9aff154a89" />


---

## 3. PowerShell Verification

After disabling the account through ADUC, Windows PowerShell was used to verify its account status.

**Execution Location:** CLIENT01-NEW  
**Shell:** Windows PowerShell

### Command 1: Verify the Disabled Account

```powershell
net user testuser01 /domain
```

**Purpose:**

Retrieves information about the `testuser01` domain account.

The command is used to check whether the account is currently active.

**Parameter explanation:**

| Parameter | Meaning |
|---|---|
| `net user` | Displays or manages user account information. |
| `testuser01` | Specifies the target username. |
| `/domain` | Queries the domain account rather than the local computer account. |

**Expected result:**

The following field should display:

```text
Account active               No
```

`No` indicates that the domain account is disabled.

**Screenshot 03: PowerShell Verification - Disabled Account**

The screenshot shows the command and its output, including `Account active: No`.



<img width="1279" height="886" alt="image" src="https://github.com/user-attachments/assets/ecbc0722-9ac8-4f37-aa22-9da1136c4314" />



---

## 4. Enabling a User Account

### Step 1: Re-enable the User Account

**Execution Location:** DC01

Open Active Directory Users and Computers.

Navigate to:

`corp.test` → `Users`

Right-click `testuser01` and select:

`Enable Account`

**Purpose:**

Enabling the account restores the account's enabled status.

This allows the user to authenticate again, provided that other authentication requirements are satisfied.

**Screenshot 04: User Account Enabled**

The screenshots show the account after it has been re-enabled.

<img width="283" height="493" alt="image" src="https://github.com/user-attachments/assets/2790b6b8-0896-47e4-ad23-723e56d92cd3" />


<img width="418" height="259" alt="image" src="https://github.com/user-attachments/assets/b4f16544-1a02-4296-8019-dd31edd8fb83" />


---

## 5. PowerShell Verification After Enabling

### Command 1: Verify the Enabled Account

**Execution Location:** CLIENT01-NEW  
**Shell:** Windows PowerShell

```powershell
net user testuser01 /domain
```

**Purpose:**

Checks the account status again after the administrator has enabled the account.

Running the same command allows the administrator to compare the account's status before and after the change.

**Parameter explanation:**

| Parameter | Meaning |
|---|---|
| `net user` | Displays user account information. |
| `testuser01` | Specifies the target username. |
| `/domain` | Queries the domain account. |

**Verified result:**

```text
Account active               Yes
```

`Yes` indicates that the domain account is enabled.

**Screenshot 05: PowerShell Verification - Enabled Account**

The screenshot shows the command and its output, including `Account active: Yes`.


<img width="1276" height="1069" alt="image" src="https://github.com/user-attachments/assets/7cf2b236-20c8-4540-979d-e98568285236" />


---

## 6. Results

The Active Directory user account was successfully disabled and re-enabled.

The following tasks were completed:

- Disabled the `testuser01` account using ADUC.
- Verified the disabled status using `net user`.
- Confirmed that `Account active` displayed `No`.
- Re-enabled the account using ADUC.
- Verified the enabled status using `net user`.
- Confirmed that `Account active` displayed `Yes`.

The results confirmed that the account's enabled status could be managed through ADUC and independently checked using a Windows command-line tool.

---

## 7. Key Learning Points

### Account Disable vs. Account Lockout

Disabling an account and locking out an account are different security mechanisms.

| Feature | Account Disable | Account Lockout |
|---|---|---|
| Cause | Administrator action | Failed authentication attempts or configured security conditions |
| Control | Administrator | Account lockout policy and authentication system |
| Automatic recovery | No | Can automatically unlock after the configured duration |
| Main purpose | Suspend account access | Protect against repeated unsuccessful authentication attempts |

A disabled account remains disabled until an administrator enables it.

A locked account may become unlocked automatically depending on the domain lockout policy.

### ADUC Account Management

Active Directory Users and Computers provides a graphical interface for managing user account states.

Administrators can use it to disable and enable accounts without directly modifying directory attributes.

### Windows Command-Line Verification

The `net user` command can retrieve domain account information.

Using `/domain` allows an administrator to query a domain account from a domain-connected computer, provided the required domain connectivity and permissions are available.

### PowerShell and Active Directory Modules

The `net user` command is a Windows command-line utility and does not require the ActiveDirectory PowerShell module.

By comparison, commands such as `Get-ADUser`, `Disable-ADAccount`, and `Enable-ADAccount` are provided by the ActiveDirectory PowerShell module.

---

## 8. Conclusion

This lab provided practical experience in disabling and re-enabling an Active Directory domain user account using ADUC.

The account status was verified through Windows PowerShell using the `net user` command.

The exercise demonstrated the difference between account disabling and account lockout, as well as the importance of verifying account changes after administrative operations.
