# Active Directory User Password Reset

## 1. Overview

### Objective

This lab demonstrates how to reset an Active Directory user password using Active Directory Users and Computers (ADUC).

It also covers the first-logon password change process and PowerShell verification of the user's password-related account attributes.

### Environment

| Item | Configuration |
|---|---|
| Server OS | Windows Server 2025 |
| Domain Controller | DC01 |
| Client OS | Windows 11 |
| Client Machine | CLIENT01 |
| Domain Name | corp.test |
| Management Tool | Active Directory Users and Computers (ADUC) |
| PowerShell Module | ActiveDirectory |

### Scenario

**Help Desk Ticket #1002**

James Wilson reported that he had forgotten his Windows domain password and could no longer sign in.

The administrator was required to reset his password and allow him to establish a new password at his next logon.

| Attribute | Value |
|---|---|
| User | James Wilson |
| SamAccountName | jwilson |
| Issue | Forgotten password |
| Requested Action | Password Reset |

---

## 2. Understanding Password Reset vs Account Unlock

Before performing the operation, it is important to understand the difference between a password reset and an account unlock.

| Operation | Purpose |
|---|---|
| Password Reset | Replaces the existing password with a new password. |
| Account Unlock | Removes the account lockout state so the user can sign in again. |

A password reset does not automatically mean that an account is unlocked.

Similarly, unlocking an account does not change the user's password.

In this scenario, James had forgotten his password, so a password reset was required.

---

## 3. Resetting the User Password

### Step 1: Locate the User Account

**Execution Location:** DC01

Open Active Directory Users and Computers.

Navigate to:

`corp.test` → `Users`

Locate the account:

`James Wilson`

**Purpose:**

The administrator must identify the correct user account before performing a password reset.

In a real Help Desk environment, the administrator should also follow the organization's identity verification and authorization procedures before making account changes.



<img width="1132" height="790" alt="image" src="https://github.com/user-attachments/assets/277b6401-5d3f-4fc9-9077-d99feb29b460" />


### Step 2: Open the Reset Password Dialog

Right-click James Wilson and select:

`Reset Password...`

This opens the password reset dialog.

**Purpose:**

This function allows an authorized administrator to assign a new password to the selected Active Directory user.



<img width="1129" height="796" alt="image" src="https://github.com/user-attachments/assets/7aadff75-5481-498c-a652-d36f872e975a" />



### Step 3: Configure the New Password

Enter a new temporary password in the following fields:

| Field | Purpose |
|---|---|
| New password | Specifies the new temporary password. |
| Confirm password | Confirms the new password by requiring it to be entered again. |

Configure the following options:

| Option | Configuration |
|---|---|
| User must change password at next logon | Checked |
| Unlock the user's account | Unchecked |

**Why these settings matter:**

**User must change password at next logon**

This requires the user to change the administrator-provided temporary password when signing in for the first time after the reset.

This helps prevent the temporary password from remaining in use.

**Unlock the user's account**

This option was left unchecked because the scenario involved a forgotten password, not an account lockout.

**Security Note:**

Temporary passwords should be communicated through an approved secure channel. Passwords must not be included in GitHub documentation, screenshots, or other public records.



<img width="573" height="388" alt="image" src="https://github.com/user-attachments/assets/4175d829-8ff7-4fcc-a6c3-519dedb6a220" />




After configuring the options, click OK to perform the reset.

---

## 4. First Logon and Password Change

### Step 4: Sign In as the User

**Execution Location:** CLIENT01

After the administrator reset the password, the user was signed in from the Windows 11 client machine.

Use the following account:

| Field | Value |
|---|---|
| Username | corp\jwilson |
| Password | Administrator-provided temporary password |

The temporary password is not documented in this repository.

**Purpose:**

This step verifies that the user can proceed through the first-logon password change process after an administrator-initiated reset.

### Step 5: Change the Password at First Logon

Because the administrator enabled `User must change password at next logon`, Windows displayed a password change screen.

James Wilson then established a new password.

After completing the password change, the user was able to proceed with the Windows sign-in process.

**Why this matters:**

The administrator provides a temporary password, but the user establishes their own password for subsequent authentication.

This is a common account provisioning and password recovery practice in enterprise IT environments.



<img width="531" height="256" alt="image" src="https://github.com/user-attachments/assets/be8a8d79-913d-4c96-8401-0fb89735b630" />


---

## 5. PowerShell Verification

After completing the password change process, PowerShell was used on the domain controller to inspect the user's account state.

**Execution Location:** DC01  
**Shell:** Windows PowerShell

### Command 1: Initial Password and Account State Check

```powershell
Get-ADUser -Identity jwilson -Properties PasswordLastSet,PasswordExpired,LockedOut |
Select-Object Name,PasswordLastSet,PasswordExpired,LockedOut
```

**Purpose:**

This command was initially used to inspect the user's password-related attributes and lockout state.

However, the command returned an error:

`Get-ADUser: One or more properties are invalid.`

The issue was caused by including `PasswordExpired` in the additional property list supplied to `Get-ADUser`.

This was corrected in the subsequent verification command.

### Command 2: Verify Password and Lockout Information

```powershell
Get-ADUser -Identity jwilson -Properties PasswordLastSet,LockedOut |
Select-Object Name,PasswordLastSet,LockedOut
```

**Purpose:**

Retrieves the user's password last-set timestamp and account lockout state.

**Parameter explanation:**

| Parameter | Meaning |
|---|---|
| `Get-ADUser` | Retrieves an Active Directory user object. |
| `-Identity jwilson` | Specifies the target user account. |
| `-Properties` | Requests additional attributes. |
| `PasswordLastSet` | Retrieves the date and time when the password was last set. |
| `LockedOut` | Retrieves whether the account is currently locked out. |
| `Select-Object` | Selects the properties to display. |
| `\|` | Passes the output to the next PowerShell command. |

**Verified Output:**

| Attribute | Observed Result | Meaning |
|---|---|---|
| Name | James Wilson | Target user account |
| PasswordLastSet | 11/17/2026 10:13:33 AM | Last password-set timestamp |
| LockedOut | False | The account was not locked out |

The output confirmed that the account had a recorded password-set timestamp and was not locked out.

**Important Note:**

`PasswordLastSet` indicates when the password was last set. It does not independently prove that the user successfully signed in.

The successful user-side password change and Windows sign-in were separately observed on CLIENT01.


Server-Side


<img width="1599" height="292" alt="image" src="https://github.com/user-attachments/assets/e641a3d8-0506-458e-83d8-f563ddddb59f" />



Client-Side

<img width="664" height="750" alt="image" src="https://github.com/user-attachments/assets/c2bb2d72-1b7b-41c9-8a96-dcadb1239219" />
<img width="573" height="640" alt="image" src="https://github.com/user-attachments/assets/e4c0fa2a-87f9-4942-ad7a-c2db07286e7c" />
<img width="505" height="679" alt="image" src="https://github.com/user-attachments/assets/6917e924-b6ea-400c-8827-0692c7e57220" />
<img width="511" height="669" alt="image" src="https://github.com/user-attachments/assets/92bd81b8-568a-4267-958a-aea65bc8e550" />



---

## 6. Results

The password reset procedure was successfully completed.

The following tasks were performed:

- Located James Wilson's Active Directory account.
- Opened the Reset Password dialog in ADUC.
- Set a new temporary password.
- Enabled `User must change password at next logon`.
- Signed in from CLIENT01 using the user's account.
- Completed the first-logon password change.
- Verified the password-set timestamp using PowerShell.
- Confirmed that the account was not locked out.

---

## 7. Key Learning Points

### Password Reset

Administrators can reset a user's password through ADUC when the user cannot access their account because they have forgotten their password.

### First-Logon Password Change

The `User must change password at next logon` option requires the user to establish a new password at the next applicable sign-in.

### Account Lockout

Password reset and account unlock are separate administrative operations.

An administrator should determine whether the problem is caused by an incorrect or forgotten password, an account lockout, or another authentication issue.

### PowerShell Administration

PowerShell provides a way to inspect Active Directory account attributes.

The `Get-ADUser` cmdlet can be used to retrieve user information and verify account states.

---

## 8. Conclusion

This lab provided practical experience in performing an Active Directory password reset using ADUC.

It also demonstrated the first-logon password change process from the user's perspective and the use of PowerShell to inspect password-related account attributes.

The exercise reinforced the distinction between password resets and account unlocks, which is important when troubleshooting authentication issues in enterprise IT environments.
