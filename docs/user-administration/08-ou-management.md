# Active Directory OU Management

## 1. Overview

This lab demonstrates how to create an Organizational Unit (OU) in Active Directory, move a user account into the OU, and verify the user's location using PowerShell.

It also introduces the optional **Protect object from accidental deletion** setting to help prevent unintended deletion of the OU.

## 2. Lab Environment

| Component | Configuration |
|---|---|
| Domain Controller | DC01 |
| Client Machine | CLIENT01-NEW |
| Operating System | Windows Server 2025 / Windows 11 |
| Domain | corp.test |
| Test User | testuser01 |
| New OU | HelpDesk-Users |

## 3. Scenario

**Ticket #008 – User Organisation Management**

A Help Desk administrator needs to organise user accounts by creating a dedicated OU and moving an existing user account into it.

The administrator must also verify that the user is located in the correct OU.

## 4. What is an Organizational Unit (OU)?

An Organizational Unit (OU) is a container within Active Directory used to organise objects such as:

- User accounts
- Groups
- Computer accounts
- Other OUs

OUs help administrators organise directory objects, apply Group Policy Objects (GPOs), and delegate administrative responsibilities.

For example, an organisation might create separate OUs for Help Desk, Finance, and Human Resources.

**Note:** The default `Users` container in Active Directory is not an OU. Unlike an OU, it cannot have a GPO linked directly to it.

## 5. Procedure

### Step 1 – Create a New OU

**Execution Location:** DC01

1. Open Active Directory Users and Computers (ADUC).
2. Expand the `corp.test` domain.
3. Right-click the domain.
4. Select `New` → `Organizational Unit`.
5. Enter `HelpDesk-Users` as the OU name.
6. Keep `Protect container from accidental deletion` enabled.
7. Click `OK`.

The new OU is created directly under the `corp.test` domain.

**Screenshot:**

<img width="1126" height="793" alt="image" src="https://github.com/user-attachments/assets/39ad881c-ace2-475f-b030-de885e725bfa" />


<img width="1125" height="798" alt="image" src="https://github.com/user-attachments/assets/4b4af974-8445-4f83-8b5c-3a6c8017d2ff" />


<img width="1129" height="798" alt="image" src="https://github.com/user-attachments/assets/dec72394-bc05-4a2b-ac69-c69fa27e2b2d" />


<img width="1126" height="796" alt="image" src="https://github.com/user-attachments/assets/9fd620c3-0070-4be5-9f7e-be1c850078ff" />



### Step 2 – Move the User Account into the OU

**Execution Location:** DC01

1. Navigate to the default `Users` container.
2. Locate `testuser01`.
3. Right-click the account and select `Move`.
4. Select `HelpDesk-Users`.
5. Confirm the operation.

The user account is now organised under the newly created OU.

**Screenshot:**

<img width="1129" height="801" alt="image" src="https://github.com/user-attachments/assets/36952e0b-a4c9-4841-9b0f-9966bb156c71" />


<img width="484" height="498" alt="image" src="https://github.com/user-attachments/assets/4ed2b154-c402-49a0-a087-458d0d09b721" />

An image showing that testuser01 has been moved from 'Users'
<img width="1129" height="802" alt="image" src="https://github.com/user-attachments/assets/fe452495-f944-4480-932e-290600c46ad4" />

An image showing that testuser01 has been moved to HelpDesk-Users
<img width="1132" height="796" alt="image" src="https://github.com/user-attachments/assets/000f5624-f2d9-4032-8950-773382fc2fa3" />


### Step 3 – Verify the OU Using PowerShell

**Execution Location:** DC01

Run the following command:

```powershell
Get-ADOrganizationalUnit -Identity "OU=HelpDesk-Users,DC=corp,DC=test"
```

**Screenshot:**

<img width="1600" height="598" alt="image" src="https://github.com/user-attachments/assets/1b2fe769-9ca5-4680-a657-20a9a1214769" />


**Purpose:**

This command retrieves the specified Organizational Unit from Active Directory.

**Command breakdown:**

- `Get-ADOrganizationalUnit`: Retrieves an OU object from Active Directory.
- `-Identity`: Specifies the OU to retrieve.
- `OU=HelpDesk-Users,DC=corp,DC=test`: The Distinguished Name (DN) of the target OU.

#### Understanding the Distinguished Name (DN)

A Distinguished Name uniquely identifies an object within the Active Directory directory hierarchy.

The DN used in this command is:

```text
OU=HelpDesk-Users,DC=corp,DC=test
```

| Component | Meaning |
|---|---|
| `OU=` | Identifies an Organizational Unit |
| `HelpDesk-Users` | Name of the OU |
| `DC=` | Identifies a Domain Component |
| `corp` | First component of the domain name |
| `test` | Second component of the domain name |

The domain `corp.test` is represented as `DC=corp,DC=test`.

Each component is separated by a comma, representing the object's position in the directory hierarchy.

### Step 4 – Verify the User's OU Location

**Execution Location:** DC01

Run the following command:

```powershell
Get-ADUser testuser01 | Select-Object Name,DistinguishedName
```

**Purpose:**

This command retrieves the user account and displays its name and full Distinguished Name.

**Command breakdown:**

- `Get-ADUser testuser01`: Retrieves the user account identified by its `sAMAccountName`.
- `|`: Passes the retrieved user object to the next command through the PowerShell pipeline.
- `Select-Object`: Selects specific properties for display.
- `Name`: Displays the user's name.
- `DistinguishedName`: Displays the full directory path of the user.

Expected output:

```text
Name       DistinguishedName
----       -----------------
Test Test  CN=Test Test,OU=HelpDesk-Users,DC=corp,DC=test
```

**Screenshot:**

<img width="1279" height="271" alt="image" src="https://github.com/user-attachments/assets/fac30bb2-9471-4f31-97f6-04c8d95dcce6" />


**Understanding the User's DN:**

- `CN=Test Test`: The Common Name of the user object.
- `OU=HelpDesk-Users`: The OU containing the user.
- `DC=corp,DC=test`: The domain components.

The output confirms that `testuser01` is located inside `HelpDesk-Users`.

## 6. Optional Security Configuration – Protect from Accidental Deletion

When creating the OU, the option `Protect container from accidental deletion` was enabled.

This setting helps prevent administrators from accidentally deleting the OU.

### How it works

Active Directory uses Access Control Lists (ACLs) to manage permissions.

- **ACL (Access Control List):** A collection of access control rules.
- **DACL (Discretionary Access Control List):** Controls which security principals are allowed or denied access.
- **ACE (Access Control Entry):** An individual permission rule within an ACL.

The accidental deletion protection setting adds deletion-denying access control entries to the OU's permissions.

These entries help prevent deletion-related operations, including deleting the OU itself or deleting its subtree.

This is a protection mechanism rather than an absolute restriction. An administrator with sufficient permissions can change the protection setting or modify the relevant permissions.

### Verify the Protection Setting

**Execution Location:** DC01

Run:

```powershell
Get-ADOrganizationalUnit -Identity "OU=HelpDesk-Users,DC=corp,DC=test" -Properties ProtectedFromAccidentalDeletion | Select-Object Name,ProtectedFromAccidentalDeletion
```

**Command breakdown:**

- `Get-ADOrganizationalUnit`: Retrieves the specified OU.
- `-Identity`: Identifies the target OU using its DN.
- `-Properties ProtectedFromAccidentalDeletion`: Requests the accidental deletion protection attribute.
- `Select-Object`: Displays only the selected properties.

Expected output:

```text
Name          ProtectedFromAccidentalDeletion
----          --------------------------------
HelpDesk-Users True
```

The `True` value confirms that accidental deletion protection is enabled.

**Screenshot:**


<img width="794" height="119" alt="image" src="https://github.com/user-attachments/assets/dfdc0a46-ecaa-48cd-84df-e27dd4e6364d" />



## 7. Verification Results

| Verification Item | Expected Result | Status |
|---|---|---|
| OU creation | HelpDesk-Users exists | Passed |
| User organisation | testuser01 is inside HelpDesk-Users | Passed |
| OU identification | OU retrieved using its DN | Passed |
| User location | DN contains OU=HelpDesk-Users | Passed |
| Accidental deletion protection | ProtectedFromAccidentalDeletion = True | Passed |

## 8. Key Findings

- Organizational Units provide a structured way to organise Active Directory objects.
- OUs support Group Policy management and delegated administration.
- Distinguished Names identify objects by their full directory hierarchy.
- PowerShell can be used to retrieve OU information and verify user account locations.
- The accidental deletion protection setting provides an additional safeguard against unintended OU deletion.
- ACLs contain ACEs, which define individual access permissions.

## 9. Conclusion

This lab demonstrated how to create an Organizational Unit, move an existing user account into it, and verify the directory structure using PowerShell.

It also introduced the relationship between ACLs, DACLs, and ACEs through the optional accidental deletion protection setting.
