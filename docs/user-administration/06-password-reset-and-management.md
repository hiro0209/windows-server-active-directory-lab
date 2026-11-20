# Password Reset and Account Password Management

## 1. Overview

This lab demonstrates how to reset an Active Directory user's password and require the user to change it at their next logon.

This simulates a common Help Desk scenario in which an employee forgets their password and requests assistance from the IT support team.

### Lab Environment

| Item | Value |
|---|---|
| Domain Controller | DC01 |
| Client Machine | CLIENT01-NEW |
| Domain | corp.test |
| Test User | testuser01 |
| Management Tool | Active Directory Users and Computers (ADUC) |
| Verification Tool | PowerShell |

### Scenario

**Ticket #006 – User Cannot Remember Their Password**

A user reports that they cannot remember their current password and requests assistance.

As a Help Desk technician, the administrator must:

1. Reset the user's password.
2. Require the user to change the temporary password at their next logon.
3. Verify that the old password is no longer valid.
4. Confirm that the user can change their password and log in successfully.

---

## 2. Reset the User's Password

### Objective

Reset the password for `testuser01` using Active Directory Users and Computers.

### Execution Location

**DC01 – Windows Server 2025**

### Procedure

1. Open **Server Manager**.
2. Select **Tools → Active Directory Users and Computers**.
3. Navigate to:

   `corp.test → Users`

4. Right-click `testuser01`.
5. Select **Reset Password...**.

**Screenshot – Password Reset Dialog**


<img width="1129" height="796" alt="image" src="https://github.com/user-attachments/assets/0fcd81af-39a1-4155-8e5a-3173faa79a8a" />


### Configure the Password Reset

In the Reset Password dialog:

1. Enter a temporary password in **New password**.
2. Re-enter the same password in **Confirm password**.
3. Select **User must change password at next logon**.
4. Leave **Unlock the user's account** unchecked.
5. Click **OK**.


<img width="1129" height="798" alt="image" src="https://github.com/user-attachments/assets/bf9dbe4d-b6a3-4748-bb40-df96a729af79" />


<img width="1132" height="795" alt="image" src="https://github.com/user-attachments/assets/49159023-6c1f-46f6-8729-b6ad24a93724" />



### Explanation of the Options

| Option | Purpose |
|---|---|
| User must change password at next logon | Requires the user to change the temporary password during their next logon. |
| Unlock the user's account | Unlocks an account that has been locked out due to authentication failures. |

The unlock option was not required because the test account was not locked out.

**Security note:** Temporary passwords should be unique, securely communicated, and never reused across other systems.

---

## 3. Verify That the Old Password Is Invalid

### Objective

Confirm that the previous password cannot authenticate after the administrator resets the account password.

### Execution Location

**CLIENT01-NEW – Windows 11**

### Procedure

1. Return to the Windows sign-in screen.
2. Select **Other user**.
3. Enter the domain username:

   `corp\testuser01`

4. Attempt to sign in using the previous password.

### Expected Result

Windows rejects the old password and displays an incorrect-password message.


<img width="526" height="649" alt="image" src="https://github.com/user-attachments/assets/b0caf8a0-a967-4106-83cf-85484d0bd81b" />



### Explanation

When an administrator resets an Active Directory password, the previous password is replaced.

Therefore, the old password should no longer authenticate the user.

This test verifies that the password reset was applied.

---

## 4. Require a Password Change at Next Logon

### Objective

Verify that the user is required to change the temporary password before accessing Windows.

### Execution Location

**CLIENT01-NEW – Windows 11**

### Procedure

1. Select **Other user**.
2. Enter:

   `corp\testuser01`

3. Enter the temporary password configured by the administrator.
4. Continue the sign-in process.

### Expected Result

Windows requires the user to change their password before completing the sign-in process.


Screenshot1:
<img width="579" height="691" alt="image" src="https://github.com/user-attachments/assets/bc29069a-c093-48c9-bc8a-af071a77e7d7" />





Screenshot2:
<img width="483" height="852" alt="image" src="https://github.com/user-attachments/assets/0f7132b9-5989-4328-9862-19aa5cb72826" />






Screenshot3:
<img width="487" height="612" alt="image" src="https://github.com/user-attachments/assets/c7f1affb-3e5f-42cd-b67c-77d4df1fea42" />




### Explanation

The **User must change password at next logon** option ensures that the administrator-provided temporary password is not intended to remain the user's permanent password.

The user must provide a new password that complies with the domain password policy.

---

## 5. Change the Password and Sign In

### Objective

Allow the user to replace the temporary password with a new password and verify successful authentication.

### Procedure

On the Windows password change screen:

1. Enter the temporary password as the old password.
2. Enter a new password.
3. Confirm the new password.
4. Submit the password change.

After the password change succeeds, Windows completes the sign-in process.

### Expected Result

The user successfully accesses the Windows desktop.

<img width="1759" height="1446" alt="image" src="https://github.com/user-attachments/assets/848146a8-ee57-4700-a884-4ee675a9164f" />



The active account can be verified using PowerShell.

---

## 6. Verify the Logged-in User

### Execution Location

**CLIENT01-NEW – PowerShell**

### Command

```powershell
whoami
```

### Command Explanation

| Component | Meaning |
|---|---|
| `whoami` | Displays the identity of the currently logged-in user. |

### Expected Output

```text
corp\testuser01
```

<img width="679" height="262" alt="image" src="https://github.com/user-attachments/assets/ada5fd00-ea1e-4e80-bff9-ab588e260f90" />



This confirms that the session is running under the intended domain account.

---

## 7. Verify the Password Last Set Attribute

### Objective

Confirm that Active Directory records the updated password change timestamp.

### Execution Location

**DC01 – PowerShell**

### Command

```powershell
Get-ADUser testuser01 -Properties PasswordLastSet | Format-List SamAccountName,PasswordLastSet
```

### Command Explanation

| Component | Meaning |
|---|---|
| `Get-ADUser` | Retrieves an Active Directory user account. |
| `testuser01` | Specifies the target account. |
| `-Properties PasswordLastSet` | Requests the password last-set timestamp. |
| `Format-List` | Displays the selected properties in a vertical list. |
| `SamAccountName` | Displays the user's logon account name. |
| `PasswordLastSet` | Displays when the password was last changed. |

### Expected Output

```text
SamAccountName : testuser01
PasswordLastSet : [Date and time of the latest password change]
```


<img width="1597" height="394" alt="image" src="https://github.com/user-attachments/assets/701c9734-fea5-4552-8ebb-66f4c7fcf304" />


The timestamp should reflect the user's successful password change.

This provides additional evidence that the password update was recorded in Active Directory.

---

## 8. Lab Results

| Test | Expected Result | Status |
|---|---|---|
| Administrator resets the password | Password reset succeeds | Completed |
| Old password authentication | Authentication is rejected | Completed |
| Temporary password authentication | Password change is required | Completed |
| User changes their password | Password change succeeds | Completed |
| User signs in again | Windows desktop is accessible | Completed |
| `whoami` verification | `corp\testuser01` | Completed |
| `PasswordLastSet` verification | Updated timestamp is displayed | Completed |

---

## 9. Key Takeaways

Through this lab, I practised the following Active Directory administration tasks:

- Resetting a domain user's password through ADUC.
- Understanding the difference between password reset and account unlocking.
- Enforcing a password change at the next logon.
- Verifying authentication failure using an old password.
- Confirming successful authentication using the updated password.
- Checking the `PasswordLastSet` attribute using PowerShell.

### Help Desk Relevance

Password reset requests are common in IT support environments.

A Help Desk technician should verify the user's identity according to organisational procedures before resetting a password. Temporary credentials should be handled securely, and the user should be encouraged to establish a new password immediately.

This lab demonstrates the technical workflow for handling a basic Active Directory password reset request.
