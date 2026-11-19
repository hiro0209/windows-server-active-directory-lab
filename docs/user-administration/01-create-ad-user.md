# Active Directory User Creation

## 1. Overview

### Objective

This lab demonstrates how to create a new user account in Active Directory Domain Services (AD DS) using the Active Directory Users and Computers (ADUC) graphical interface.

After creating the account, PowerShell commands are used to verify the user's information and identify important Active Directory attributes.

### Environment

| Item | Configuration |
|---|---|
| Server OS | Windows Server 2025 |
| Domain Controller | DC01 |
| Domain Name | corp.test |
| Management Tool | Active Directory Users and Computers (ADUC) |
| PowerShell Module | ActiveDirectory |

### Scenario

A new employee, James Wilson, has joined the IT department.

An administrator needs to create his domain account and configure the appropriate user information.

| Attribute | Value |
|---|---|
| First Name | James |
| Last Name | Wilson |
| Full Name | James Wilson |
| User Logon Name | jwilson |
| Department | IT |
| Description | IT Department Employee |
| Job Title | IT Support Officer |
| Company | Contoso Ltd |

---

## 2. Creating a User Account

### Step 1: Open Active Directory Users and Computers

**Execution Location:** DC01

Open Active Directory Users and Computers (ADUC).

Navigate to:

`corp.test` → `Users`

This is the default Users container in the domain.

**Purpose:**

ADUC is a graphical management tool used to administer Active Directory objects, including users, groups, and computers.

<img width="1129" height="796" alt="image" src="https://github.com/user-attachments/assets/e59cd736-da00-4abc-a47b-d05932d960c9" />



<img width="1129" height="837" alt="image" src="https://github.com/user-attachments/assets/9b0311d9-738b-47ea-8547-f85beeb9ab6e" />



### Step 2: Create a New User

Right-click the Users container and select:

`New` → `User`

Enter the following information:

| Field | Value |
|---|---|
| First Name | James |
| Last Name | Wilson |
| Full Name | James Wilson |
| User Logon Name | jwilson |

The User Logon Name is the account name used when signing in to the domain.

<img width="652" height="567" alt="image" src="https://github.com/user-attachments/assets/53411bda-3632-4d98-9d93-5e501d84c769" />


<img width="654" height="568" alt="image" src="https://github.com/user-attachments/assets/01e6ba85-5b56-45c1-bb0b-26160d0d597e" />


### Step 3: Configure the Initial Password

Set an initial password for the new account.

The following account options were configured:

| Option | Configuration |
|---|---|
| User must change password at next logon | Checked |
| User cannot change password | Unchecked |
| Password never expires | Unchecked |
| Account is disabled | Unchecked |

**Why these settings matter:**

- **User must change password at next logon:** Requires the user to replace the administrator-provided initial password when signing in.
- **User cannot change password:** Left unchecked so the user can manage their own password.
- **Password never expires:** Left unchecked to avoid disabling the configured password-expiration policy.
- **Account is disabled:** Left unchecked so the account can be enabled for use.



<img width="657" height="570" alt="image" src="https://github.com/user-attachments/assets/094c5801-b2b7-4e1a-bf24-7e82e1b02643" />


<img width="655" height="562" alt="image" src="https://github.com/user-attachments/assets/f75345bf-93fe-448c-88a7-18e614d93b9c" />



### Step 4: Configure Additional User Information

After creating the account, open the user's Properties window.

Under the **General** tab, configure:

| Attribute | Value |
|---|---|
| Description | IT Department Employee |

Under the **Organization** tab, configure:

| Attribute | Value |
|---|---|
| Job Title | IT Support Officer |
| Department | IT |
| Company | Contoso Ltd |
| Manager | Not configured |
| Direct Reports | Not configured |

**Purpose:**

These attributes provide additional information about the employee.

In a real organization, this information can help administrators identify users, understand their organizational roles, and support directory searches and administrative processes.


<img width="1129" height="799" alt="image" src="https://github.com/user-attachments/assets/9f9826e0-1a1e-4a24-b546-ea67bc439154" />


<img width="1129" height="799" alt="image" src="https://github.com/user-attachments/assets/620c3b04-68cd-457b-b48b-868fcc1e2ec7" />



<img width="615" height="805" alt="image" src="https://github.com/user-attachments/assets/f76e65d4-b847-4904-8322-fd4d0d50c9d0" />




<img width="612" height="808" alt="image" src="https://github.com/user-attachments/assets/5876da12-e73e-46dd-9e6c-4ed63b031a4d" />




### Step 5: Verify the Created User in ADUC

Return to the Users container.

Confirm that James Wilson appears in the list of domain users.


<img width="1128" height="793" alt="image" src="https://github.com/user-attachments/assets/50236947-b175-412c-94ea-05a7c8fc0ec8" />


---

## 3. PowerShell Verification

After creating the account through ADUC, PowerShell was used to verify that the account existed and that its attributes were configured correctly.

**Execution Location:** DC01  
**Shell:** Windows PowerShell

### Command 1: Verify the User Account

```powershell
Get-ADUser -Identity jwilson
```

**Purpose:**

Retrieves the Active Directory user object identified by `jwilson`.

**Parameter explanation:**

| Parameter | Meaning |
|---|---|
| `Get-ADUser` | Retrieves an Active Directory user account. |
| `-Identity` | Specifies the user account to retrieve. |
| `jwilson` | The SamAccountName used to identify the account. |

**Expected result:**

PowerShell displays the user object for James Wilson, confirming that the account can be retrieved from Active Directory.

<img width="954" height="586" alt="image" src="https://github.com/user-attachments/assets/8e0eaa0f-fb59-4e5c-a327-3cedc5880727" />


### Command 2: Verify User Attributes

```powershell
Get-ADUser -Identity jwilson -Properties Department,Description,Title,Company |
Select-Object Name,SamAccountName,Department,Description,Title,Company
```

**Purpose:**

Retrieves additional user attributes and displays selected properties in a readable format.

**Parameter explanation:**

| Parameter | Meaning |
|---|---|
| `-Identity jwilson` | Identifies the target user account. |
| `-Properties` | Requests additional attributes that are not included in the default property set. |
| `Department` | Retrieves the user's department. |
| `Description` | Retrieves the account description. |
| `Title` | Retrieves the user's job title. |
| `Company` | Retrieves the company name. |
| `Select-Object` | Selects the properties to display. |
| `\|` | Passes the output of the first command to the next command. |

**Verified output:**

| Attribute | Verified Value |
|---|---|
| Name | James Wilson |
| SamAccountName | jwilson |
| Department | IT |
| Description | IT Department Employee |
| Title | IT Support Officer |
| Company | Contoso Ltd |

The output confirmed that the configured user attributes were correctly stored in Active Directory.


<img width="1560" height="427" alt="image" src="https://github.com/user-attachments/assets/36525525-7518-46c0-a3b4-c165635a1d24" />



### Command 3: Verify Important Account Identifiers

```powershell
Get-ADUser -Identity jwilson -Properties UserPrincipalName |
Select-Object Name,SamAccountName,UserPrincipalName,SID,DistinguishedName
```

**Purpose:**

Retrieves important identifiers associated with the Active Directory user account.

**Parameter explanation:**

| Parameter | Meaning |
|---|---|
| `-Properties UserPrincipalName` | Requests the UserPrincipalName attribute. |
| `Select-Object` | Displays the selected account identifiers. |
| `SID` | Security Identifier associated with the account. |
| `DistinguishedName` | LDAP distinguished name identifying the object's location in Active Directory. |

**Important Active Directory identifiers:**

| Identifier | Explanation |
|---|---|
| Name | The display name of the user object. |
| SamAccountName | The traditional Windows account logon name. |
| UserPrincipalName (UPN) | A user logon identifier commonly formatted like an email address. |
| SID | A unique security identifier used by Windows for security and access control. |
| DistinguishedName (DN) | Identifies the object's location within the Active Directory directory structure. |



<img width="1588" height="1123" alt="image" src="https://github.com/user-attachments/assets/f8c8a72f-bba4-413a-8764-dce21ccc61fc" />




**Key technical concepts:**

- The SamAccountName is commonly used for traditional domain logons.
- The UPN provides an alternative logon format, usually `username@domain`.
- The UPN format does not necessarily mean that the user has a matching email address.
- The SID is used by Windows security mechanisms to identify the security principal. Changing the account name does not normally change its SID.
- The Distinguished Name identifies the object's position in the directory hierarchy.

---

## 4. Results

The Active Directory user account was successfully created and verified.

The following tasks were completed:

- Created a new domain user account using ADUC.
- Configured the initial password and account options.
- Added user information, including Description, Job Title, Department, and Company.
- Confirmed that the account appeared in the domain's Users container.
- Used PowerShell to retrieve and verify the user account.
- Verified the configured user attributes.
- Retrieved important identifiers, including SamAccountName, UPN, SID, and DistinguishedName.

The results confirmed that the user account and its configured attributes were available in Active Directory.

---

## 5. Key Learning Points

### Active Directory User Management

ADUC provides a graphical interface for creating and managing domain user accounts.

### PowerShell Administration

PowerShell allows administrators to retrieve and inspect user account information efficiently.

### Active Directory Attributes

User attributes provide information about an employee's identity, department, and organizational role.

### Security Identifiers

The SID is particularly important because Windows uses it to identify security principals when applying permissions and access control.

---

## 6. Conclusion

This lab provided practical experience in creating an Active Directory user account using ADUC and verifying the account through PowerShell.

It also introduced important Active Directory concepts, including user attributes, SamAccountName, UPN, SID, and Distinguished Name.
