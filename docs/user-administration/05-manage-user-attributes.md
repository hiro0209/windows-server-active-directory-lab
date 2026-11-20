Manage Active Directory User Attributes

## 1. Overview

### Objective

This lab demonstrates how to view, modify, and verify user attributes in Active Directory using Active Directory Users and Computers (ADUC) and PowerShell.

The objective is to understand how administrators maintain user information and verify that changes have been successfully saved.

### Environment

| Item | Configuration |
|---|---|
| Server | Windows Server 2025 |
| Domain Controller | DC01 |
| Client | CLIENT01-NEW |
| Domain | corp.test |
| Test User | testuser01 |
| GUI Tool | Active Directory Users and Computers |
| Command-line Tool | Windows PowerShell |

### Scenario

A user’s department and contact information need to be updated. The administrator modifies the user's attributes in Active Directory and verifies the updated information using PowerShell.

---

## 2. View Existing User Attributes

**Execution Location:** DC01

### Procedure

1. Open Server Manager.
2. Select **Tools → Active Directory Users and Computers**.
3. Expand the `corp.test` domain.
4. Open the `Users` container.
5. Right-click `testuser01`.
6. Select **Properties**.
7. Open the **General** tab.
8. Review the existing user attributes.

### Purpose

Before making changes, administrators should review the existing information to understand which attributes need to be updated.

### Screenshot

<img width="1128" height="871" alt="image" src="https://github.com/user-attachments/assets/e418be57-a36f-4df3-a91f-b8945924e5d9" />

---

## 3. Modify User Attributes

**Execution Location:** DC01

### Procedure

In the user's Properties window, update the following attributes:

| Attribute | Updated Value |
|---|---|
| First name | Hiro |
| Last name | Test |
| Display name | Hiro Test |
| Description | Help Desk Lab Test Account |
| Office | Adelaide |
| Telephone number | 0400000000 |
| E-mail | hiro.test@corp.test |

1. Enter the updated information.
2. Select **Apply**.
3. Select **OK** to save the changes.

### Purpose

Active Directory user attributes store information associated with user accounts.

Administrators may update these attributes when employees change departments, job titles, office locations, or contact details.

### Screenshot


<img width="1128" height="868" alt="image" src="https://github.com/user-attachments/assets/29c8a93d-f8b4-419b-80c8-c6eaa5572668" />


---

## 4. Verify Updated Attributes Using GUI

**Execution Location:** DC01

### Procedure

1. Reopen the Properties window for `testuser01`.
2. Select the **General** tab.
3. Confirm that the updated information has been saved.

### Expected Result

The updated attributes should match the values entered in the previous step.

### Screenshot



<img width="1126" height="862" alt="image" src="https://github.com/user-attachments/assets/b387342c-97a6-49b1-84a2-ec673b5a3947" />



---

## 5. Verify User Attributes Using PowerShell

**Execution Location:** DC01

### Step 1: Check Active Directory PowerShell Module

Check whether the ActiveDirectory module is available:

```powershell
Get-Module -ListAvailable ActiveDirectory
```

Check whether the `Get-ADUser` command is available:

```powershell
Get-Command Get-ADUser
```

Import the module if necessary:

```powershell
Import-Module ActiveDirectory
```

### Purpose

The ActiveDirectory PowerShell module provides cmdlets for managing and querying Active Directory objects.

`Get-ADUser` is one of the cmdlets provided by this module.

### Step 2: Retrieve User Information

Execute:

```powershell
Get-ADUser testuser01 -Properties *
```

### Command Explanation

| Parameter | Explanation |
|---|---|
| `Get-ADUser` | Retrieves an Active Directory user account |
| `testuser01` | Specifies the target user |
| `-Properties *` | Requests all additional user attributes |

### Step 3: Display Selected Attributes

To display only the attributes relevant to this lab, execute:

```powershell
Get-ADUser testuser01 -Properties * | Format-List GivenName,Surname,DisplayName,Description,Office,TelephoneNumber,EmailAddress
```

### Command Explanation

| Command | Explanation |
|---|---|
| `Get-ADUser` | Retrieves the AD user object |
| `-Properties *` | Retrieves additional attributes |
| `\|` | Passes the output to the next command |
| `Format-List` | Displays selected properties in a vertical list |

### Expected Result

The following attributes should display the updated values:

| Attribute | Expected Value |
|---|---|
| GivenName | Hiro |
| Surname | Test |
| DisplayName | Hiro Test |
| Description | Help Desk Lab Test Account |
| Office | Adelaide |
| TelephoneNumber | 0400000000 |
| EmailAddress | hiro.test@corp.test |

### Screenshot



<img width="1591" height="498" alt="image" src="https://github.com/user-attachments/assets/8a701cdf-3e7f-49d7-be07-7d1b6707efed" />



---

## 6. Results

The user attributes of `testuser01` were successfully modified through Active Directory Users and Computers.

The updated information was verified through both the graphical interface and PowerShell using `Get-ADUser`.

This demonstrated two methods of inspecting Active Directory user information.

---

## 7. Key Learning Points

- Active Directory stores user information as attributes.
- ADUC provides a graphical interface for modifying user attributes.
- PowerShell provides a command-line method for retrieving user information.
- `Get-ADUser` retrieves Active Directory user objects.
- The `-Properties` parameter allows administrators to retrieve additional attributes.
- `Format-List` helps display selected attributes in a readable format.
- GUI and PowerShell verification provide complementary ways to confirm changes.

---

## 8. Conclusion

This lab demonstrated how to modify and verify Active Directory user attributes using ADUC and PowerShell.
