# IP Configuration and DHCP Verification

## 1. Overview

This document describes the process of examining IP configuration, DHCP lease information, routing, DNS settings, and basic network connectivity on a Windows 11 client.

The objective was to understand the network configuration and verify basic network functionality using Windows command-line tools.

## 2. Lab Environment

| Device | Operating System | Hostname |
|---|---|---|
| Client | Windows 11 | CLIENT01-NEW |
| Domain Controller | Windows Server 2025 | DC01 |

**Domain:** `corp.test`

CLIENT01-NEW had two network adapters configured:

| Adapter | IPv4 Address | DHCP Server |
|---|---|---|
| Ethernet | `192.168.56.105` | `192.168.56.100` |
| Ethernet 2 | `10.0.3.15` | `10.0.3.2` |

## 3. IP Configuration Verification

### Step 1. Examine Network Configuration

**Execution location:** CLIENT01-NEW

**Command:**

```powershell
ipconfig /all
```

**Purpose:**

Displays detailed network configuration for all network adapters.

**Why this command was used:**

To identify the IP addresses, subnet masks, DHCP settings, DNS servers, and default gateways assigned to the client.

**Key findings:**

- Ethernet IPv4 address: `192.168.56.105`
- Ethernet 2 IPv4 address: `10.0.3.15`
- DHCP was enabled on both adapters.
- DHCP servers were identified as `192.168.56.100` and `10.0.3.2`.

This confirmed that both network adapters had DHCP-enabled IPv4 configurations.

Screenshot1:

<img width="999" height="822" alt="image" src="https://github.com/user-attachments/assets/db45f5c6-6baf-41e0-8e33-6450b6952168" />


Screenshot2:


<img width="994" height="562" alt="image" src="https://github.com/user-attachments/assets/2783a31c-3ea7-4adb-89b8-6f978168d81c" />


## 4. Network Connectivity Verification

### Step 2. Test External IP Connectivity

**Execution location:** CLIENT01-NEW

**Command:**

```powershell
ping 8.8.8.8
```

**Purpose:**

Tests IPv4 connectivity to an external IP address.

**Why this command was used:**

To check whether the client could communicate with an external IP address without relying on DNS name resolution.

**Result:**

The ping test returned successful replies.

This confirmed IPv4 reachability to `8.8.8.8` at the time of testing.


Screenshot3:


<img width="1023" height="931" alt="image" src="https://github.com/user-attachments/assets/648830f7-330d-45c7-a6c2-3540157dc28b" />


### Step 3. Examine the IPv4 Routing Table

**Execution location:** CLIENT01-NEW

**Command:**

```powershell
route print -4
```

**Purpose:**

Displays the IPv4 routing table.

**Why this command was used:**

To identify the available routes and understand how Windows forwards packets to local and external destinations.

**Key findings:**

The default route was:

```text
0.0.0.0/0 via 10.0.3.2
```

The default gateway was `10.0.3.2`.

Screenshot4:



<img width="895" height="741" alt="image" src="https://github.com/user-attachments/assets/1a571537-1566-4b70-ba8b-7034c37c2cd0" />



**Technical explanation:**

The default route is used when no more specific route matches the destination IP address.

Windows uses the routing table and the longest prefix match rule to select the appropriate route.

## 5. DNS Verification

### Step 4. Perform a DNS Lookup

**Execution location:** CLIENT01-NEW

**Command:**

```powershell
nslookup google.com
```

**Purpose:**

Queries the configured DNS server to resolve a domain name into IP address information.

**Why this command was used:**

To examine DNS name resolution and identify which DNS server the client was using.

**Observed output:**

```text
Server: Unknown
Address: 192.168.56.10
```

The DNS server address was `192.168.56.10`, which corresponds to DC01.

Screenshot5:


<img width="622" height="282" alt="image" src="https://github.com/user-attachments/assets/32f38ca1-fda6-489a-9136-f4cb671e45d4" />




**Technical explanation:**

The `Server: Unknown` message indicates that the DNS server's hostname could not be identified by the lookup.

It does not, by itself, prove that DNS name resolution is failing.

### Step 5. Perform a Reverse DNS Lookup

**Execution location:** CLIENT01-NEW

**Command:**

```powershell
nslookup 192.168.56.10
```

**Purpose:**

Performs a reverse DNS lookup to identify the hostname associated with an IP address.

**Why this command was used:**

To investigate why the DNS server was displayed as `Unknown` during the previous lookup.

**Result:**

```text
Unknown
can't find 192.168.56.10: Non-existent domain
```

Screenshot6:


<img width="667" height="130" alt="image" src="https://github.com/user-attachments/assets/8ed94ff7-a6f2-4b61-846d-ce5ae5236300" />



**Technical explanation:**

The reverse lookup returned a Non-existent domain (NXDOMAIN) response.

This indicates that the requested reverse DNS record could not be found.

A missing PTR record or an absent/incomplete reverse lookup zone could explain this result.

Further DNS investigation would be required to confirm the exact cause.

## 6. DHCP Lease Verification

### Step 6. Renew DHCP Leases

**Execution location:** CLIENT01-NEW

**Command:**

```powershell
ipconfig /renew
```

**Purpose:**

Requests renewal of DHCP leases for network adapters configured to obtain IP addresses automatically.

**Why this command was used:**

To verify that the client could renew its DHCP leases and continue using dynamically assigned network configurations.

**Result:**

Both network adapters retained their existing IP addresses:

| Adapter | Previous IP | IP After Renewal |
|---|---|---|
| Ethernet | `192.168.56.105` | `192.168.56.105` |
| Ethernet 2 | `10.0.3.15` | `10.0.3.15` |

The lease expiration times were extended after renewal.


Screenshot7(Before):

<img width="954" height="448" alt="image" src="https://github.com/user-attachments/assets/b0c5eb3f-6b91-403f-9b02-8eb2757abd3c" />

Secreenshot8(After):

<img width="954" height="448" alt="image" src="https://github.com/user-attachments/assets/cfa7a52e-9b20-4831-8403-aa8072a6c1d2" />


Screenshot9(Before):

<img width="994" height="561" alt="image" src="https://github.com/user-attachments/assets/aced8f23-45d1-4705-bb9e-c763030cd076" />


Screenshot10(After):

<img width="994" height="561" alt="image" src="https://github.com/user-attachments/assets/3b5fd63e-1668-47d6-b0d9-ec44327ae824" />



**Technical explanation:**

A successful DHCP lease renewal does not necessarily assign a new IP address.

The DHCP server may renew the existing lease and allow the client to continue using the same IP address.



<img width="440" height="277" alt="image" src="https://github.com/user-attachments/assets/645562bd-8c12-412b-bec9-97394f7d9caf" />


## 7. Final Results

| Verification Area | Result |
|---|---|
| IP Configuration | Examined |
| DHCP Configuration | Verified as enabled |
| External IP Connectivity | Successful |
| Default Gateway | Identified as `10.0.3.2` |
| DNS Server | Identified as `192.168.56.10` |
| Reverse DNS Lookup | Non-existent domain returned |
| DHCP Lease Renewal | Successful; existing IP addresses retained |

## 8. Key Technical Learnings

- `ipconfig /all` provides detailed information about network adapters and DHCP configuration.
- `ping` can test IP connectivity without requiring DNS name resolution.
- `route print -4` helps identify the routing paths used by Windows.
- `nslookup` can be used to examine forward and reverse DNS lookups.
- DHCP lease renewal does not necessarily change the assigned IP address.
- A DNS server displayed as `Unknown` does not automatically mean that DNS is unavailable.

## 9. Conclusion

This exercise provided practical experience in examining Windows network configurations and verifying basic network functionality.

The client successfully communicated with an external IP address and renewed its DHCP leases while retaining its existing IP addresses.

The DNS reverse lookup returned Non-existent domain, indicating an area that requires further investigation.
