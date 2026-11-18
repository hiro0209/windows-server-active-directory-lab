# 04. Network Connectivity Troubleshooting

## 1. Overview

This document describes the troubleshooting process used to establish network connectivity between a Windows Server 2025 Domain Controller (DC01) and a Windows 11 client (CLIENT01-NEW) in a VirtualBox Host-only network.

The troubleshooting process included:

- Checking IP configuration and network connectivity
- Verifying ARP entries and routing information
- Checking Windows network profiles
- Investigating Windows Defender Firewall rules
- Enabling the required ICMPv4 inbound firewall rule
- Verifying connectivity using `ping`

## 2. Lab Environment

| Device | Operating System | IP Address | Network |
|---|---|---|---|
| DC01 | Windows Server 2025 | `192.168.56.10` | Host-only |
| CLIENT01-NEW | Windows 11 | `192.168.56.105` | Host-only |

**Domain:** `corp.test`

Both virtual machines were connected to the same VirtualBox Host-only network.

## 3. Initial Problem

Initially, ICMP connectivity between DC01 and CLIENT01-NEW was unsuccessful.

The following tests were performed:

- CLIENT01-NEW → DC01: Ping failed
- DC01 → CLIENT01-NEW: Ping failed

The objective was to identify whether the issue was related to IP configuration, ARP resolution, routing, or Windows Defender Firewall.

## 4. Troubleshooting Process

### Step 1. Verify IP Configuration

**Execution location:** DC01 and CLIENT01-NEW

**Command:**

```powershell
ipconfig
```

**Purpose:**

Displays the network configuration of the machine, including:

- IPv4 address
- Subnet mask
- Default gateway
- Network adapter information

**Why this command was used:**

To confirm that both machines had the expected IP addresses and were connected to the intended Host-only network.

**Result:**

- DC01: `192.168.56.10`
- CLIENT01-NEW: `192.168.56.105`

Both machines had IP addresses in the `192.168.56.0` network.

### Step 2. Check ARP Table

**Execution location:** DC01

**Command:**

```powershell
arp -a
```

**Purpose:**

Displays the ARP cache, which maps IPv4 addresses to MAC addresses.

**Why this command was used:**

To check whether DC01 had learned the MAC address of CLIENT01-NEW through ARP resolution.

**Result:**

An ARP entry for CLIENT01-NEW was present in the ARP table.

The MAC address was:

```text
08-00-27-79-4B-5E
```

This indicated that DC01 had an ARP mapping for CLIENT01-NEW.

### Step 3. Check Neighbor State

**Execution location:** DC01

**Command:**

```powershell
Get-NetNeighbor -IPAddress 192.168.56.105
```

**Purpose:**

Displays neighbor information for the specified IP address, including its MAC address and reachability state.

**Why this command was used:**

To verify whether DC01 had a valid neighbor entry for CLIENT01-NEW.

**Result:**

The neighbor state was:

```text
Reachable
```

The MAC address was:

```text
08-00-27-79-4B-5E
```

This showed that the neighbor entry was considered reachable.

### Step 4. Check Routing Table

**Execution location:** DC01

**Command:**

```powershell
route print -4
```

**Purpose:**

Displays the IPv4 routing table.

**Why this command was used:**

To verify whether DC01 had a route to the `192.168.56.0` network through the correct network interface.

**Result:**

The routing table contained the following network:

```text
192.168.56.0    On-link
```

The interface IP address was:

```text
192.168.56.10
```

This confirmed that the `192.168.56.0` network was directly connected to DC01.

### Step 5. Check Windows Network Profile

**Execution location:** DC01 and CLIENT01-NEW

**Command:**

```powershell
Get-NetConnectionProfile
```

**Purpose:**

Displays the network profile assigned to each network connection.

Windows Defender Firewall uses network profiles to determine which firewall rules apply.

The main profiles are:

- Domain
- Private
- Public

**Why this command was used:**

To identify the active network profile and determine which firewall rules should apply to the Host-only network connection.

**Result:**

Both machines showed:

```text
NetworkCategory : DomainAuthenticated
```

Therefore, the Domain profile was applicable to the network connection.

### Step 6. Investigate Windows Defender Firewall

**Execution location:** CLIENT01-NEW

**GUI navigation:**

1. Open Windows Defender Firewall with Advanced Security.
2. Select Inbound Rules.
3. Locate:

```text
File and Printer Sharing (Echo Request - ICMPv4-In)
```

4. Open the rule properties.
5. Check the General tab.
6. Check the Advanced tab.

**Purpose:**

To determine whether Windows Defender Firewall was blocking incoming ICMPv4 Echo Request packets.

**Findings:**

The following configuration was identified:

| Setting | Configuration |
|---|---|
| Rule | File and Printer Sharing (Echo Request - ICMPv4-In) |
| Direction | Inbound |
| Protocol | ICMPv4 |
| Action | Allow |
| Profile | Domain |
| Enabled | Unchecked |

The `Enabled` checkbox was not selected.

In the Windows environment used for this lab, this rule was initially disabled by default.

**Important:**

The firewall rule itself must be enabled before its Allow action can permit the matching traffic.

The active network profile was `DomainAuthenticated`, so the Domain profile needed to be selected for this rule.

### Step 7. Enable the ICMPv4 Inbound Firewall Rule

**Execution location:** CLIENT01-NEW

**Actions:**

1. Open the properties of:

```text
File and Printer Sharing (Echo Request - ICMPv4-In)
```

2. In the General tab, check `Enabled`.
3. Confirm that the Action is set to `Allow`.
4. In the Advanced tab, ensure that `Domain` is selected.
5. Apply the changes.

**Purpose:**

To allow incoming ICMPv4 Echo Request packets on the Domain network profile.

**Why this change was necessary:**

CLIENT01-NEW was using the `DomainAuthenticated` network profile.

The required inbound firewall rule was disabled, meaning that incoming ICMPv4 Echo Request traffic was not being permitted by this rule.

Enabling the rule allowed the firewall to apply its Allow action to matching incoming ICMPv4 traffic.

### Step 8. Verify Connectivity

**Execution location:** Both machines

**Command from CLIENT01-NEW:**

```powershell
ping 192.168.56.10
```

**Purpose:**

Tests ICMP connectivity from CLIENT01-NEW to DC01.

**Command from DC01:**

```powershell
ping 192.168.56.105
```

**Purpose:**

Tests ICMP connectivity from DC01 to CLIENT01-NEW.

**Result:**

Both directions were successful.

| Source | Destination | Result |
|---|---|---|
| CLIENT01-NEW | DC01 | Successful |
| DC01 | CLIENT01-NEW | Successful |

Network connectivity between the two virtual machines was established.

## 5. Final Results

The troubleshooting process successfully established network connectivity between DC01 and CLIENT01-NEW.

| Area | Final Result |
|---|---|
| IP configuration | Verified |
| ARP resolution | Neighbor entry found |
| Routing | Directly connected network confirmed |
| Network profile | DomainAuthenticated |
| Firewall | Domain ICMPv4 inbound rule enabled |
| Ping | Successful in both directions |

## 6. Key Technical Learnings

### 7.1 Windows Defender Firewall Profiles

Windows Defender Firewall applies rules according to the active network profile.

A rule configured for the Private profile does not automatically apply to a DomainAuthenticated connection.

It is important to verify the active network profile before modifying firewall rules.

### 7.2 Enabled and Allow Are Different Settings

Two important firewall settings must be understood:

- **Enabled:** Determines whether the firewall rule is active.
- **Action = Allow:** Determines whether matching traffic is permitted by that rule.

A rule configured with Action = Allow will not provide the intended permission if the rule itself is disabled.

### 7.3 Inbound and Outbound Traffic

The rule used in this lab was:

```text
File and Printer Sharing (Echo Request - ICMPv4-In)
```

Inbound means traffic entering the local machine.

When DC01 sends a Ping request to CLIENT01-NEW, CLIENT01-NEW receives an ICMPv4 Echo Request.

Therefore, the inbound firewall rule on CLIENT01-NEW is relevant to allowing that request.

---

## 8. Conclusion

The initial connectivity issue was resolved by enabling the Domain profile's ICMPv4 inbound firewall rule on CLIENT01-NEW.

The `Enabled` checkbox had initially been unchecked, even though the rule's action was configured as `Allow`.

After enabling the rule and applying the Domain profile, Ping communication was successful in both directions.

Wireshark and Npcap were then installed on CLIENT01-NEW, allowing network traffic to be captured directly from its Host-only Ethernet adapter.

This completed the network connectivity troubleshooting stage and prepared the lab environment for further Kerberos packet analysis.
