# DNS Name Resolution Troubleshooting

## 1. Overview

This document describes the investigation and resolution of DNS name resolution issues in a Windows Server 2025 and Windows 11 lab environment.

The investigation focused on:

- Examining the client's DNS configuration.
- Testing external and internal DNS name resolution.
- Investigating the `Server: Unknown` message.
- Identifying a missing Reverse Lookup Zone and PTR record.
- Creating a Reverse Lookup Zone and registering a PTR record.
- Verifying forward and reverse DNS resolution.

## 2. Lab Environment

| Device | Operating System | Hostname | IP Address |
|---|---|---|---|
| Client | Windows 11 | CLIENT01-NEW | `192.168.56.105` |
| Domain Controller / DNS Server | Windows Server 2025 | DC01 | `192.168.56.10` |

**Domain:** `corp.test`

### Network Configuration

CLIENT01-NEW uses two network adapters.

| Adapter | IPv4 Address | Default Gateway | Purpose |
|---|---|---|---|
| Ethernet | `192.168.56.105` | Not configured | Host-only communication |
| Ethernet 2 | `10.0.3.15` | `10.0.3.2` | External network communication |

The Host-only adapter communicates with the internal lab network, while the NAT adapter provides external network connectivity.

The absence of a default gateway on the Host-only adapter is expected in this configuration because communication with devices in the same subnet does not require a default gateway.

**DNS Server:** `192.168.56.10` (DC01)

## 3. Initial DNS Investigation

### Step 1. Examine Client DNS Configuration

**Execution location:** CLIENT01-NEW

**Command:**

```powershell
ipconfig /all
```

**Purpose:**

Displays detailed network configuration, including IP addresses, DHCP settings, DNS servers, and default gateways.

**Why this command was used:**

To identify the DNS server configured on the client and understand the roles of its two network adapters.

**Key findings:**

- The Host-only adapter used `192.168.56.105`.
- The Host-only adapter did not have a default gateway configured.
- The NAT adapter used `10.0.3.15`.
- The NAT adapter used `10.0.3.2` as its default gateway.
- DC01 (`192.168.56.10`) was the DNS server used for the lab environment.

**Technical explanation:**

A default gateway is used to forward traffic to destinations outside the local network when no more specific route is available.

The Host-only adapter communicates with DC01 through the directly connected `192.168.56.0/24` network. Therefore, it does not require its own default gateway.

External traffic is forwarded through the NAT adapter's default gateway (`10.0.3.2`).

**Screenshot 1: Client Network Configuration**

![Client Network Configuration](../../screenshots/troubleshooting/03-01-client-network-configuration.png)

### Step 2. Test External DNS Resolution

**Execution location:** CLIENT01-NEW

**Command:**

```powershell
nslookup google.com
```

**Purpose:**

Queries the configured DNS server to resolve the external domain `google.com`.

**Why this command was used:**

To verify whether the client could resolve an external domain through its configured DNS server.

**Observed output:**

The DNS server was displayed as:

```text
Server: Unknown
Address: 192.168.56.10
```

An initial DNS request timeout was also observed:

```text
DNS request timed out.
Timeout was 2 seconds.
```

However, the IP address information for `google.com` was eventually returned.

When the command was tested again later, the timeout was no longer observed.

**Result:**

External DNS name resolution was successful.

**Technical explanation:**

The `Server: Unknown` message does not necessarily indicate that the DNS server is unavailable.

It means that the DNS server's hostname could not be identified for display.

The initial timeout indicated that a DNS request did not receive a response within the two-second period. Since the requested domain was eventually resolved, the lookup itself ultimately succeeded.

The reason for the initial timeout could not be confirmed. The issue was not reproduced during the later test.

**Screenshot 2: External DNS Resolution**

![External DNS Resolution](../../screenshots/troubleshooting/03-02-external-dns-resolution.png)

### Step 3. Test Internal DNS Resolution

**Execution location:** CLIENT01-NEW

**Command:**

```powershell
nslookup dc01.corp.test
```

**Purpose:**

Tests whether the client's DNS server can resolve the internal hostname of the Domain Controller.

**Why this command was used:**

To verify internal DNS name resolution within the `corp.test` domain.

**Result:**

The DNS server was still displayed as `Unknown`, but no timeout message was observed.

A further test was performed by explicitly specifying the DNS server.

**Command:**

```powershell
nslookup dc01.corp.test 192.168.56.10
```

**Purpose:**

Queries DC01 directly instead of relying on the client's default DNS server selection.

**Result:**

The hostname was successfully resolved to:

```text
dc01.corp.test
192.168.56.10
```

This confirmed that the internal forward DNS lookup was working correctly.

**Screenshot 3: Internal DNS Resolution**

![Internal DNS Resolution](../../screenshots/troubleshooting/03-03-internal-dns-resolution.png)

**Screenshot 4: Internal DNS Query to Specified Server**

![Internal DNS Query](../../screenshots/troubleshooting/03-04-internal-dns-query.png)

## 4. Reverse DNS Investigation

### Step 4. Perform a Reverse DNS Lookup

**Execution location:** CLIENT01-NEW

**Command:**

```powershell
nslookup 192.168.56.10
```

**Purpose:**

Performs a reverse DNS lookup to identify the hostname associated with an IP address.

**Why this command was used:**

To investigate why the DNS server was displayed as `Unknown`.

**Result:**

```text
Server: Unknown
Address: 192.168.56.10

Unknown can't find 192.168.56.10:
Non-existent domain
```

The query returned an NXDOMAIN response.

**Technical explanation:**

A reverse DNS lookup uses a PTR (Pointer) record to map an IP address to a hostname.

The lookup failed because the corresponding reverse DNS record could not be found.

Further investigation was required to identify whether the Reverse Lookup Zone or the PTR record was missing.

**Screenshot 5: Initial Reverse DNS Lookup Failure**

![Reverse DNS Lookup Failure](../../screenshots/troubleshooting/03-05-reverse-dns-lookup.png)

### Step 5. Examine DNS Reverse Lookup Zones

**Execution location:** DC01

**GUI navigation:**

1. Open Server Manager.
2. Select Tools.
3. Open DNS.
4. Expand the DNS server.
5. Examine Reverse Lookup Zones.

**Purpose:**

To determine whether a Reverse Lookup Zone existed for the `192.168.56.0/24` network.

**Result:**

No Reverse Lookup Zone was present.

This explained why the reverse lookup could not find the corresponding PTR record.

**Screenshot 6: DNS Reverse Lookup Zones Before Configuration**

![Reverse Lookup Zones Before Configuration](../../screenshots/troubleshooting/03-06-dns-reverse-lookup-zones-before.png)

## 5. DNS Configuration and Resolution

### Step 6. Create a Reverse Lookup Zone

**Execution location:** DC01

**GUI navigation:**

1. Open DNS Manager.
2. Right-click Reverse Lookup Zones.
3. Select New Zone.
4. Select Primary Zone.
5. Enable Store the zone in Active Directory.
6. Select the replication scope for DNS servers running on Domain Controllers in the domain.
7. Select IPv4 Reverse Lookup Zone.
8. Select Network ID and enter `192.168.56`.
9. Select Allow only secure dynamic updates.
10. Complete the wizard.

**Purpose:**

To create a Reverse Lookup Zone for the `192.168.56.0/24` network.

**Why this configuration was used:**

The lab uses an Active Directory-integrated DNS environment.

Storing the zone in Active Directory allows DNS zone information to be managed through AD-integrated DNS and replicated to other applicable Domain Controllers.

Secure dynamic updates restrict DNS record updates to authenticated and authorized clients.

**Created Reverse Lookup Zone:**

```text
56.168.192.in-addr.arpa
```

**Technical explanation:**

IPv4 reverse DNS uses the special domain:

```text
in-addr.arpa
```

The network octets are reversed when constructing the reverse lookup zone name.

For the `192.168.56.0/24` network:

```text
192.168.56
        ↓
56.168.192.in-addr.arpa
```

**Result:**

The Reverse Lookup Zone was successfully created.

Its initial records included:

- SOA (Start of Authority)
- NS (Name Server)

However, no PTR record for DC01 was present.

**Screenshot 7: Reverse Lookup Zone Created**

![Reverse Lookup Zone Created](../../screenshots/troubleshooting/03-07-reverse-lookup-zone-created.png)

### Step 7. Register a PTR Record

**Execution location:** DC01

**GUI navigation:**

1. Expand Reverse Lookup Zones.
2. Select `56.168.192.in-addr.arpa`.
3. Right-click the zone.
4. Select New Pointer (PTR).
5. Enter the following information.

**PTR Record Configuration:**

| Field | Value |
|---|---|
| Host IP Address | `10` |
| Host name | `dc01.corp.test` |

The option allowing any authenticated user to update all DNS records with the same name was left unchecked.

**Purpose:**

To register a PTR record that maps DC01's IP address to its hostname.

**Why this configuration was used:**

The Reverse Lookup Zone already represented the `192.168.56.0/24` network.

Therefore, only the final IP address octet (`10`) was required.

The resulting PTR record maps:

```text
192.168.56.10
        ↓
dc01.corp.test
```

**Result:**

The PTR record was successfully created.

**Screenshot 8: PTR Record Created**

![PTR Record Created](../../screenshots/troubleshooting/03-08-ptr-record-created.png)

### Step 8. Verify Reverse DNS Resolution

**Execution location:** CLIENT01-NEW

**Command:**

```powershell
nslookup 192.168.56.10
```

**Purpose:**

To verify whether the newly registered PTR record allows the client to resolve the IP address into its corresponding hostname.

**Result:**

The reverse lookup successfully returned:

```text
dc01.corp.test
192.168.56.10
```

The previous NXDOMAIN response was resolved.

This confirmed that reverse DNS resolution was now working correctly.

## 6. Final Verification Results

| Verification | Initial Result | Final Result |
|---|---|---|
| External DNS lookup (`google.com`) | Initial timeout, then successful resolution | Successful; timeout not reproduced |
| Internal DNS lookup (`dc01.corp.test`) | Successful | Successful |
| Reverse DNS lookup (`192.168.56.10`) | NXDOMAIN | Successful |
| Reverse Lookup Zone | Not configured | Created |
| PTR Record | Missing | Registered |
| DNS Server display | Unknown | Hostname identifiable after PTR registration |

The initial DNS timeout was not reproduced during the later test. Its underlying cause was not established.

## 7. Key Technical Learnings

### 7.1 Forward DNS Lookup

Forward DNS lookup resolves a hostname into an IP address.

For example:

```text
dc01.corp.test
        ↓
192.168.56.10
```

The DNS record commonly used for IPv4 forward resolution is an A record.

### 7.2 Reverse DNS Lookup

Reverse DNS lookup resolves an IP address into a hostname.

For example:

```text
192.168.56.10
        ↓
dc01.corp.test
```

The DNS record used for this purpose is a PTR record.

### 7.3 A Record vs PTR Record

| Record | Purpose | Example |
|---|---|---|
| A | Hostname to IPv4 address | `dc01.corp.test` → `192.168.56.10` |
| PTR | IPv4 address to hostname | `192.168.56.10` → `dc01.corp.test` |

A and PTR records perform different functions. Successful forward resolution does not automatically guarantee successful reverse resolution.

### 7.4 Forward Lookup Zone vs Reverse Lookup Zone

**Forward Lookup Zone**

Stores records used to resolve hostnames into IP addresses.

Example:

```text
corp.test
```

**Reverse Lookup Zone**

Stores records used to resolve IP addresses into hostnames.

Example:

```text
56.168.192.in-addr.arpa
```

### 7.5 What Does Server: Unknown Mean?

When `nslookup` starts, it displays the configured DNS server.

For example:

```text
Server: Unknown
Address: 192.168.56.10
```

The DNS server's IP address is known, but its hostname could not be identified.

A missing PTR record for the DNS server's IP address can cause this display.

It does not automatically mean that the DNS server is unavailable or that forward DNS resolution has failed.

### 7.6 What Does NXDOMAIN Mean?

NXDOMAIN stands for Non-Existent Domain.

It is a DNS response indicating that the queried domain name does not exist in the DNS namespace being queried.

In this investigation, the reverse lookup returned NXDOMAIN because the corresponding PTR record could not be found.

### 7.7 What Does DNS Request Timed Out Mean?

A DNS request timeout occurs when a DNS response is not received within the configured waiting period.

In this lab, an initial two-second timeout was observed during the external DNS lookup.

However, the requested domain was eventually resolved, and the timeout was not reproduced during later testing.

A timeout should therefore be investigated separately from NXDOMAIN and `Server: Unknown`.

### 7.8 SOA and NS Records

When the Reverse Lookup Zone was created, two records were initially visible.

| Record | Full Name | Purpose |
|---|---|---|
| SOA | Start of Authority | Contains authoritative information about the DNS zone, including its primary server and administrative parameters. |
| NS | Name Server | Identifies the DNS servers authoritative for the zone. |

These records are created as part of the DNS zone structure. They do not replace PTR records.

### 7.9 Why Was the Host-only Default Gateway Not Configured?

The Host-only adapter did not require a default gateway because DC01 and CLIENT01-NEW communicated within the same directly connected subnet.

The NAT adapter provided the default gateway (`10.0.3.2`) for external network communication.

A missing default gateway on one adapter is not automatically a configuration error. The routing design and intended network communication must be considered.

## 8. Conclusion

The investigation identified that the DNS environment could successfully resolve external and internal hostnames, but reverse DNS resolution for DC01 initially failed.

The root cause was the absence of a Reverse Lookup Zone and the corresponding PTR record.

A Reverse Lookup Zone for `192.168.56.0/24` was created, and a PTR record was registered for DC01.

After the configuration change, reverse DNS resolution successfully returned `dc01.corp.test` for `192.168.56.10`.

This exercise provided practical experience in DNS troubleshooting, forward and reverse name resolution, DNS record management, and verification of DNS configuration in an Active Directory environment.
