# Active Directory Security Group Management

## 1. Overview

This lab demonstrates how to create and manage a Security Group in Active Directory, add a user to the group, verify group membership using PowerShell, and remove the user.

Security Groups are used to manage access to shared resources such as folders, files, printers, and other network resources. Instead of assigning permissions to individual users, administrators can assign permissions to groups and manage access through group membership.

## 2. Lab Environment

| Item | Value |
|---|---|
| Domain Controller | DC01 |
| Operating System | Windows Server 2025 |
| Domain Name | corp.test |
| Client Account | testuser01 |
| Group Name | HelpDesk-Support |
| Group Scope | Global |
| Group Type | Security |

## 3. Scenario

**Ticket #007 – User Access Management**

A user requires membership in the HelpDesk-Support security group.

As a Help Desk technician, the task is to:

1. Create a Security Group named `HelpDesk-Support`.
2. Configure the group scope as Global.
3. Configure the group type as Security.
4. Add `testuser01` to the group.
5. Verify group membership using PowerShell.
6. Remove `testuser01` from the group.
7. Verify that the user has been removed.

## 4. Understanding Group Scope and Group Type

Before creating a group, it is important to understand the available Group Scope and Group Type options.

### 4.1 Group Scope

Group Scope determines which users and groups can become members of a group and where that group can be used to assign permissions.

| Group Scope | Description | Typical Usage |
|---|---|---|
| Domain Local | Can contain accounts and groups from trusted domains. Primarily used to assign permissions to resources within its own domain. | Assigning permissions to a shared folder or printer in a domain. |
| Global | Organises users and groups from the same domain according to roles or responsibilities. Can be added to Domain Local or Universal groups in trusted domains. | Creating role-based groups such as HelpDesk-Support or Finance-Team. |
| Universal | Can contain accounts and Global groups from any domain in the forest. Can be used to assign permissions across multiple domains. | Managing access across a multi-domain Active Directory forest. |

**Practical example:**

- Global: `HelpDesk-Support` – identifies users who belong to the Help Desk team.
- Domain Local: `DL-SharedFolder-Modify` – represents users who are allowed to modify a particular shared folder.
- Universal: `UG-IT-Department` – combines role groups across multiple domains when organisation-wide membership is required.

Universal group membership is stored in the Global Catalog. Changes to Universal group membership can therefore have replication implications in a multi-domain forest.

### 4.2 Group Type

Group Type determines how the group is used.

| Group Type | Description | Typical Usage |
|---|---|---|
| Security | Can be assigned permissions to access resources. | Controlling access to folders, files, printers, and other secured resources. |
| Distribution | Used for email distribution lists. It cannot be used to assign Windows resource permissions. | Sending emails to a department or team through a distribution list. |

**Important distinction:**

- Security Groups are used for access control.
- Distribution Groups are used for email distribution.

A Security Group can also be mail-enabled in supported messaging environments.

### 4.3 Common Group Management Strategy: AGDLP

A common Active Directory permission management model is AGDLP:

- **A – Accounts:** Individual user accounts.
- **G – Global Groups:** Organise users according to their roles.
- **DL – Domain Local Groups:** Represent permissions for specific resources.
- **P – Permissions:** Assign permissions to the Domain Local Group.

Example:

`testuser01` → `HelpDesk-Support` → `DL-SharedFolder-Modify` → Modify permission

This approach separates organisational roles from resource permissions, making access management easier to maintain.

## 5. Procedure

### Step 1 – Open Active Directory Users and Computers

On DC01:

1. Open Server Manager.
2. Select Tools.
3. Open Active Directory Users and Computers.
4. Expand `corp.test`.
5. Select the `Users` container.


Screenshot1:

<img width="1128" height="796" alt="image" src="https://github.com/user-attachments/assets/d281484f-e0f4-4477-9e0b-e054e8bb38fd" />


Screenshot2:

<img width="1129" height="838" alt="image" src="https://github.com/user-attachments/assets/ccbbcf37-a23a-4e69-baf0-f92eb3f42f38" />






### Step 2 – Create a New Security Group

Right-click the `Users` container.

1. Select New.
2. Select Group.
3. Enter the following information:

| Setting | Value |
|---|---|
| Group name | HelpDesk-Support |
| Group scope | Global |
| Group type | Security |

4. Click OK.

Screenshot3:


<img width="1129" height="796" alt="image" src="https://github.com/user-attachments/assets/f5add18f-f6d0-4db7-8f77-041b2574ba81" />


Screenshot4:


<img width="1131" height="798" alt="image" src="https://github.com/user-attachments/assets/1810d895-bc07-445b-adf3-3546a7eeaa87" />



### Step 3 – Verify Group Creation

Confirm that `HelpDesk-Support` appears in the `Users` container.

Screenshot5:

<img width="1129" height="882" alt="image" src="https://github.com/user-attachments/assets/bdf26c71-ecd2-4543-be0a-f4dd2194dff0" />


Screenshot6:

<img width="1128" height="799" alt="image" src="https://github.com/user-attachments/assets/eb49113b-cba8-4738-8b92-f1e5c5ad4540" />



### Step 4 – Open Group Properties

1. Right-click `HelpDesk-Support`.
2. Select Properties.
3. Open the Members tab.

Initially, the group should have no members.

Screenshot7:


<img width="1129" height="796" alt="image" src="https://github.com/user-attachments/assets/8e1204e8-777b-4c0e-8022-b475a5f775cb" />



### Step 5 – Add a User to the Security Group

1. In the Members tab, click Add.
2. Enter `testuser01`.
3. Click Check Names.
4. Confirm that the account is resolved.
5. Click OK.
6. Click Apply to save the changes.

Screenshot8:

<img width="699" height="423" alt="image" src="https://github.com/user-attachments/assets/bb769267-7b34-424a-8d50-cc07c3008ff8" />


Screenshot9:

<img width="687" height="382" alt="image" src="https://github.com/user-attachments/assets/9615a604-0222-4bc8-868f-7f50dc159eab" />



Screenshot10:

<img width="699" height="424" alt="image" src="https://github.com/user-attachments/assets/519a307f-8c2e-465d-b3ec-36e39efa84a1" />


Screenshot11:

<img width="598" height="682" alt="image" src="https://github.com/user-attachments/assets/d1911421-e0d2-4901-bdf4-ecb5c76f0dde" />


**Purpose:**

Adding a user to a Security Group establishes group membership. The user can then receive permissions assigned to that group, subject to the applicable resource permissions and access controls.

Membership alone does not automatically grant access to a resource. The group must also be assigned the appropriate permissions on that resource.

### Step 6 – Verify Group Membership Using PowerShell

**Execution location:** DC01 – Windows PowerShell

Run the following command:

```powershell
Get-ADGroupMember -Identity "HelpDesk-Support" | Format-Table Name,SamAccountName,ObjectClass
```

**Command explanation:**

| Command / Parameter | Description |
|---|---|
| `Get-ADGroupMember` | Retrieves the members of an Active Directory group. |
| `-Identity` | Specifies the target group. |
| `"HelpDesk-Support"` | Identifies the group to query. |
| `\|` | Passes the command output to the next command. |
| `Format-Table` | Displays the results in a table format. |
| `Name` | Displays the object's name. |
| `SamAccountName` | Displays the account's logon name. |
| `ObjectClass` | Identifies the object type, such as user or group. |

**Expected result:**

The output should show `testuser01` as a member of `HelpDesk-Support`.

Screenshot12:


<img width="1594" height="297" alt="image" src="https://github.com/user-attachments/assets/70e08751-5154-4a0d-92e6-a34367918da8" />


### Step 7 – Remove a User from the Security Group

To test access administration and removal procedures:

1. Open `HelpDesk-Support` Properties.
2. Select the Members tab.
3. Select `testuser01`.
4. Click Remove.
5. Confirm the deletion when prompted.
6. Click Apply.

Screenshot13:


<img width="1129" height="880" alt="image" src="https://github.com/user-attachments/assets/407ba6ba-c0cd-47cf-82b7-c1084e686e4c" />

Screenshot14:

<img width="601" height="684" alt="image" src="https://github.com/user-attachments/assets/38f96c03-715d-4968-9cda-b7119c46eef2" />


Screenshot15:

<img width="763" height="685" alt="image" src="https://github.com/user-attachments/assets/77de86c0-e5ac-4841-8b01-0e5bc29f3ae1" />


Screenshot16:

<img width="600" height="685" alt="image" src="https://github.com/user-attachments/assets/3d7cb6f2-4172-4883-b49e-8a74ef1ebbae" />


**Purpose:**

Removing a user from a Security Group prevents that user from receiving permissions through that group.

In a real Help Desk environment, this may be necessary when an employee changes departments, no longer requires access, or leaves the organisation.

### Step 8 – Verify Group Membership Removal

**Execution location:** DC01 – Windows PowerShell

Run the following command:

```powershell
Get-ADGroupMember -Identity "HelpDesk-Support"
```

**Expected result:**

The command executes successfully but returns no member objects.

This confirms that the group currently has no members.

The group itself still exists in Active Directory.

Screenshot17:

<img width="1279" height="151" alt="image" src="https://github.com/user-attachments/assets/8b08d234-0233-474d-b620-1a13ae8904d5" />


## 6. Verification Results

| Test | Expected Result | Status |
|---|---|---|
| Create Security Group | HelpDesk-Support created | Passed |
| Configure Group Scope | Global | Passed |
| Configure Group Type | Security | Passed |
| Add User | testuser01 added successfully | Passed |
| Verify Membership | testuser01 displayed by PowerShell | Passed |
| Remove User | testuser01 removed successfully | Passed |
| Verify Removal | No member objects returned | Passed |

## 7. Key Findings

Through this lab, I successfully demonstrated and verified the following Active Directory administration tasks:

- Created a Security Group named `HelpDesk-Support`.
- Configured the group scope as Global.
- Configured the group type as Security.
- Added `testuser01` to the group using Active Directory Users and Computers.
- Verified group membership using `Get-ADGroupMember`.
- Removed `testuser01` from the group.
- Confirmed that the group contained no members after removal.

This lab demonstrated how Security Groups can simplify access management by organising users according to their roles rather than assigning permissions individually.

It also highlighted the importance of verifying both access assignment and access removal during Help Desk operations.

## 8. Conclusion

Understanding the differences between Group Scope and Group Type helps administrators design appropriate group structures and manage access securely.
