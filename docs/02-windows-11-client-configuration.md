# Windows 11 Client Configuration and Active Directory Domain Integration

## 1. Overview

This document describes the installation and configuration of a Windows 11 client virtual machine and its integration into an existing Active Directory domain.

The purpose of this stage is to establish communication between a Windows 11 client and an existing Windows Server 2025 Domain Controller.

The configuration covers:

- Windows 11 virtual machine deployment
- VirtualBox Guest Additions installation
- Host-only network configuration
- IPv4 and DNS configuration
- DNS A record and SRV record verification
- Active Directory domain integration
- Domain authentication verification

## 2. Objectives

After completing this configuration, the lab should be able to:

1. Run a Windows 11 client in the same isolated virtual network as the Domain Controller.
2. Resolve the Active Directory domain name through the Domain Controller's DNS service.
3. Discover the Domain Controller using an Active Directory DNS SRV record.
4. Join the Windows 11 client to the existing Active Directory domain.
5. Authenticate using a domain account.
6. Verify the client's domain identity and Domain Controller discovery.

These steps establish the infrastructure required for subsequent authentication experiments.

## 3. Lab Environment

### 3.1 Existing Domain Controller

The Domain Controller was configured in the previous stage.

| Item | Configuration |
|---|---|
| Computer Name | DC01 |
| Operating System | Windows Server 2025 |
| Domain Name | corp.test |
| NetBIOS Domain Name | CORP |
| IPv4 Address | 192.168.56.10 |
| Main Roles | Active Directory Domain Services (AD DS), DNS |

### 3.2 Windows Client

| Item | Configuration |
|---|---|
| Virtual Machine Name | CLIENT01-NEW |
| Operating System | Windows 11 Enterprise Evaluation |
| RAM | 4096 MB |
| CPU | 2 virtual processors |
| Virtual Disk | 80 GB |
| Disk Allocation | Dynamically allocated |
| Network Adapter | Host-only Adapter |
| IPv4 Address | 192.168.56.105 |
| DNS Server | 192.168.56.10 |
| Domain | corp.test |

### 3.3 Network Architecture

Both virtual machines communicate through the VirtualBox Host-only network.

| Device | Role | IPv4 Address |
|---|---|---|
| DC01 | Domain Controller and DNS Server | 192.168.56.10 |
| CLIENT01-NEW | Domain-joined Windows client | 192.168.56.105 |

The client uses DC01 as its DNS server.

This is important because Active Directory relies on DNS to locate domain controllers and their services.

## 4. Windows 11 Client Installation

### 4.1 Create the Virtual Machine

A new virtual machine named `CLIENT01-NEW` was created in VirtualBox.

The Windows 11 Enterprise Evaluation ISO was selected as the installation media.

The following virtual hardware configuration was used:

- Base Memory: 4096 MB
- Processors: 2
- Virtual Hard Disk: 80 GB
- Disk Format: VDI
- Storage Allocation: Dynamically allocated

The Windows 11 installation was completed using the installation wizard.

### 4.2 Install VirtualBox Guest Additions

Guest Additions provides integration features between the host system and the guest operating system.

It improves usability through features such as:

- Automatic display resizing
- Improved mouse integration
- Better display driver support

Installation procedure:

1. Start CLIENT01-NEW.
2. Open the VirtualBox Devices menu.
3. Select Insert Guest Additions CD Image.
4. Open File Explorer inside Windows 11.
5. Open the mounted VirtualBox Guest Additions CD.
6. Execute `VBoxWindowsAdditions.exe`.
7. Complete the installation.
8. Restart the virtual machine.
9. Enable View → Auto-resize Guest Display.

## 5. VirtualBox Network Configuration

The client must be connected to the same virtual network as DC01.

The following settings were configured under:

**VirtualBox Manager → CLIENT01-NEW → Settings → Network → Adapter 1**

| Setting | Value |
|---|---|
| Enable Network Adapter | Enabled |
| Attached to | Host-only Adapter |
| Name | VirtualBox Host-Only Ethernet Adapter |
| Promiscuous Mode | Deny |
| Cable Connected | Enabled |

Adapter 2 was left disabled.

### Why Host-only Networking?

Host-only networking allows virtual machines and the host system to communicate through an isolated virtual network.

For this lab, it provides a network environment in which the Windows client can communicate with the Domain Controller without requiring a bridged connection to the physical LAN.

## 6. Verify the Client Network Configuration

After starting CLIENT01-NEW, PowerShell was opened with administrator privileges.

### 6.1 Check the IP Configuration

Command:

```powershell
Get-NetIPConfiguration
```

**Purpose:**

This command displays the network configuration of the computer.

It provides information such as:

- Network interface
- IPv4 address
- Default gateway
- DNS server addresses

**Why was it used?**

Before configuring DNS, the client's current network configuration needed to be checked.

This helps determine whether the network adapter is operational and whether the client has received an IPv4 address.

Observed IPv4 address:

```text
192.168.56.105
```

### 6.2 Check the DNS Server Configuration

Command:

```powershell
Get-DnsClientServerAddress -InterfaceAlias "Ethernet"
```

**Purpose:**

Displays the DNS server addresses configured for the specified network interface.

**Why was it used?**

Active Directory clients must use the DNS infrastructure that contains the domain's Active Directory service records.

The command was used to check whether CLIENT01-NEW was using DC01 as its DNS server.

The DNS server needed to be:

```text
192.168.56.10
```

## 7. Configure the DNS Server

The client DNS configuration was updated to point to DC01.

Command:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.56.10
```

**Purpose:**

Configures the DNS server address for the specified network interface.

Command breakdown:

| Component | Meaning |
|---|---|
| `Set-DnsClientServerAddress` | Changes the DNS server configuration |
| `-InterfaceAlias` | Specifies the network interface |
| `"Ethernet"` | Name of the network interface |
| `-ServerAddresses` | Specifies the DNS server address |
| `192.168.56.10` | IPv4 address of DC01 |

**Why was it used?**

The Windows client needs to query the DNS service hosted by DC01 to locate Active Directory domain services.

Using an unrelated DNS server may prevent the client from resolving internal Active Directory records.

### 7.1 Verify the DNS Configuration

Command:

```powershell
Get-DnsClientServerAddress -InterfaceAlias "Ethernet"
```

**Purpose:**

Checks whether the DNS configuration change was applied successfully.

Expected IPv4 DNS server:

```text
192.168.56.10
```

## 8. DNS Verification

DNS verification was performed before attempting domain integration.

This is important because Active Directory uses DNS to locate Domain Controllers and associated services.

### 8.1 Verify the Domain A Record

Command:

```powershell
Resolve-DnsName corp.test -Server 192.168.56.10
```

**Purpose:**

Queries the specified DNS server for the name `corp.test`.

The `Resolve-DnsName` command performs DNS name resolution.

Command breakdown:

| Component | Meaning |
|---|---|
| `Resolve-DnsName` | Performs a DNS lookup |
| `corp.test` | Domain name being queried |
| `-Server` | Specifies the DNS server to query |
| `192.168.56.10` | DC01 DNS server |

**Why was it used?**

The first step was to confirm that the client could communicate with DC01's DNS service and resolve the Active Directory domain name.

The expected A record address was:

```text
192.168.56.10
```

This confirmed that the domain name could be resolved through the specified DNS server.

### 8.2 Verify the LDAP SRV Record

Command:

```powershell
Resolve-DnsName -Name "_ldap._tcp.dc._msdcs.corp.test" -Type SRV -Server 192.168.56.10 -DnsOnly
```

**Purpose:**

Queries the DNS SRV record used to locate LDAP services provided by Active Directory Domain Controllers.

An SRV record provides service discovery information, including:

- Service name
- Protocol
- Target hostname
- Port number
- Priority
- Weight

Command breakdown:

| Component | Meaning |
|---|---|
| `Resolve-DnsName` | Performs DNS resolution |
| `-Name` | Specifies the DNS record name |
| `_ldap` | LDAP service |
| `_tcp` | TCP transport protocol |
| `_dc` | Domain Controller locator record |
| `_msdcs` | Microsoft Domain Controller service records zone |
| `corp.test` | Active Directory domain |
| `-Type SRV` | Requests an SRV record |
| `-Server` | Specifies the DNS server |
| `-DnsOnly` | Performs DNS-only resolution |

**Why was it used?**

Resolving the domain A record alone does not demonstrate that the client can discover the specific services required by Active Directory.

The LDAP SRV record helps Active Directory clients locate a Domain Controller offering LDAP services.

This verification was therefore performed before joining the client to the domain.

Observed record information:

| Item | Observed Value |
|---|---|
| Target Hostname | dc01.corp.test |
| Port | 389 |
| Priority | 0 |
| Weight | 100 |

An additional A record resolved the target hostname to:

```text
192.168.56.10
```

This confirmed that the client could discover the LDAP service endpoint through DNS.

**Important distinction:**

This test verifies LDAP service discovery through DNS. It does not directly verify Kerberos ticket exchange or Kerberos packet traffic.

## 9. Join the Windows Client to the Active Directory Domain

After confirming DNS resolution, CLIENT01-NEW was joined to the existing Active Directory domain.

Procedure:

1. Open Windows Settings.
2. Navigate to System → About.
3. Open the domain or workgroup configuration.
4. Select the option to change the computer's domain membership.
5. Select Domain.
6. Enter:

```text
corp.test
```

7. When prompted for credentials, enter the domain account:

```text
Administrator@corp.test
```

8. Enter the corresponding domain Administrator password.
9. Complete the domain join operation.
10. Restart the client.

**Purpose:**

Domain joining establishes the computer's membership in the Active Directory domain.

This allows the client to use domain identities and participate in centralized Windows domain management.

## 10. Verify Domain Authentication and Domain Controller Discovery

After restarting, the client was signed in using:

```text
Administrator@corp.test
```

The following commands were used to verify the domain integration.

### 10.1 Verify the Current User Identity

Command:

```powershell
whoami
```

**Purpose:**

Displays the current security identity of the logged-in user.

Observed result:

```text
corp\administrator
```

**Why was it used?**

This confirms that the current session is running under the CORP domain Administrator identity rather than a local Windows account.

It verifies the account context, but does not independently identify which authentication protocol was used.

### 10.2 Verify Domain Controller Discovery

Command:

```powershell
nltest /dsgetdc:corp.test
```

**Purpose:**

Requests information about a Domain Controller for the specified domain.

Command breakdown:

| Component | Meaning |
|---|---|
| `nltest` | Windows command-line utility for Netlogon and domain operations |
| `/dsgetdc` | Requests discovery of a Domain Controller |
| `corp.test` | Target Active Directory domain |

**Why was it used?**

A successful domain login should be followed by a check that the client can locate a Domain Controller for its domain.

This command helps verify that Windows can discover the appropriate Domain Controller.

Observed result:

```text
DC: \\dc01.corp.test
Domain Name: corp.test
Forest Name: corp.test
```

This confirmed that CLIENT01-NEW successfully discovered DC01.

## 11. Troubleshooting and Diagnostic Approach

This section records the DNS-related troubleshooting performed during the client configuration process.

### 11.1 DNS Request Timeout

**Observed symptom:**

During earlier client-side DNS testing, a DNS request timeout was encountered.

Example of the reported output:

```text
DNS request timed out.
timeout was 2 seconds.
Server: Unknown
Address: 192.168...
```

**Diagnostic approach:**

A DNS timeout indicates that the DNS query did not receive a response within the configured waiting period.

It does not, by itself, identify the root cause.

The following areas were therefore checked:

1. The client's DNS server configuration.
2. The IP address of the intended DNS server.
3. The availability of the DNS service on DC01.
4. Whether the client could communicate with the intended DNS server.
5. Whether the required DNS records existed.

**Commands used:**

```powershell
Get-DnsClientServerAddress -InterfaceAlias "Ethernet"
```

This was used to identify which DNS server the client was configured to query.

```powershell
Resolve-DnsName corp.test -Server 192.168.56.10
```

This was used to query DC01 directly, avoiding ambiguity about which DNS server would receive the request.

The DNS configuration was subsequently set to DC01's address, and domain name resolution was successfully verified.

**Technical note:**

The available observations do not establish one definitive root cause for the earlier timeout. The troubleshooting record therefore distinguishes the observed symptom from the verified corrective configuration.

### 11.2 Active Directory SRV Record Verification

The client also required verification of the DNS service records used by Active Directory.

The following command was used:

```powershell
Resolve-DnsName -Name "_ldap._tcp.dc._msdcs.corp.test" -Type SRV -Server 192.168.56.10 -DnsOnly
```

This was used because a successful lookup of `corp.test` alone does not prove that the client can locate the LDAP service of a Domain Controller.

The SRV lookup ultimately returned the expected DC01 hostname and LDAP port.

This confirmed the relevant LDAP service discovery record.

## 12. Final Verification Results

The following results were confirmed after completing the configuration.

| Test | Result |
|---|---|
| Windows 11 installation | Successful |
| Guest Additions installation | Successful |
| Host-only network connection | Configured |
| Client IPv4 address | 192.168.56.105 |
| Client DNS server | 192.168.56.10 |
| Domain A record resolution | Successful |
| LDAP SRV record resolution | Successful |
| Domain membership | Successful |
| Domain account login | Successful |
| `whoami` | `corp\administrator` |
| `nltest /dsgetdc:corp.test` | DC01 discovered |

## 13. Outcome and Next Stage

CLIENT01-NEW was successfully configured as a Windows 11 domain-joined client in the existing `corp.test` Active Directory environment.

The client can:

- Communicate with the Domain Controller through the Host-only network.
- Use DC01 as its DNS server.
- Resolve the Active Directory domain name.
- Discover the LDAP service record for DC01.
- Authenticate using a domain account.
- Discover the Domain Controller through Windows domain discovery.

This completes the Windows client integration stage.
