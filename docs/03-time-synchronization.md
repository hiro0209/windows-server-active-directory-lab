# Windows Time Synchronization (NTP) Configuration and Troubleshooting

## 1. Overview

This document records the time synchronization configuration and
troubleshooting performed in the Windows Server 2025 Active Directory
lab.

The objective was to configure the Domain Controller (DC01) to obtain
time from an external Network Time Protocol (NTP) source, then verify
that the Windows 11 domain client (CLIENT01-NEW) synchronized its time
through the Active Directory domain time hierarchy.

Accurate time synchronization is important in an Active Directory
environment because Kerberos authentication uses timestamps to help
prevent replay attacks. If the time difference between a client and a
domain controller exceeds the configured Kerberos clock-skew tolerance,
authentication can fail even when DNS and domain connectivity are
working correctly.

This document records the commands that were used, why they were used,
the questions investigated during troubleshooting, and the observed
results. Where the exact command output was not captured in the
conversation, this is explicitly identified rather than reconstructed.

## 2. Objectives

-   Understand why time synchronization is required in an Active
    Directory environment.
-   Configure DC01 to use an external NTP peer.
-   Confirm that DC01 can reach and measure the external NTP source.
-   Configure the domain client to synchronize through the domain
    hierarchy.
-   Investigate time-service and Kerberos-related clock differences.
-   Record the troubleshooting process and distinguish TCP connectivity
    tests from NTP/UDP tests.
-   Verify the time difference between DC01 and CLIENT01-NEW.

## 3. Lab Environment

  Component                      Configuration
  ------------------------------ ----------------------------------
  Hypervisor                     Oracle VirtualBox 7.2.8
  Domain Controller              DC01
  Server OS                      Windows Server 2025
  Domain                         `corp.test`
  NetBIOS domain name            `CORP`
  DC01 Host-only IP              `192.168.56.10`
  DC01 NAT address               `10.0.3.15`
  Domain Client                  CLIENT01-NEW
  Client OS                      Windows 11 Enterprise Evaluation
  Client IP                      `192.168.56.105`
  Client DNS server              `192.168.56.10`
  Client time zone               `Australia/Adelaide`
  External NTP peer configured   `time.cloudflare.com`
  Windows time service           `W32Time`

### 3.1 Network Adapter Design

DC01 uses two virtual network adapters:

  -----------------------------------------------------------------------
  Adapter                 Network mode            Purpose
  ----------------------- ----------------------- -----------------------
  Adapter 1               Host-only               Private communication
                                                  between DC01 and
                                                  CLIENT01-NEW

  Adapter 2               NAT                     Outbound connectivity
                                                  from DC01 to the
                                                  external NTP service
  -----------------------------------------------------------------------

CLIENT01-NEW remains on the Host-only network. The Host-only network
provides the private lab connection, while the additional NAT adapter
gives DC01 a route to an external time source.

The NAT adapter was added without replacing the existing Host-only
adapter. This preserved the domain network configuration.

## 4. Why Time Synchronization Is Required

### 4.1 What Is NTP?

Network Time Protocol (NTP) is a protocol used to synchronize clocks
between networked systems. NTP normally uses UDP port 123.

A device configured as an NTP client sends time requests to an NTP
server and uses the responses to estimate and correct its clock. The
protocol accounts for network delay and clock offset; it is more than
simply copying the displayed time from another computer.

### 4.2 Why Does Active Directory Need Accurate Time?

Kerberos authentication uses timestamps in authentication messages. This
helps protect the authentication process against replay attacks.

If a client and a domain controller have clocks that differ by too much,
Kerberos may reject authentication requests. In a typical Windows domain
configuration, the default maximum Kerberos clock skew is five minutes
(300 seconds), although Group Policy can change this value.

The five-minute value is a tolerance limit, not a recommended
synchronization target. Systems should normally remain synchronized much
more closely.

### 4.3 Active Directory Time Hierarchy

In a typical Active Directory domain:

1.  The forest-root domain's PDC Emulator is the authoritative time
    source for the domain hierarchy.
2.  Other domain controllers synchronize according to the domain time
    hierarchy.
3.  Domain-joined clients normally use the domain hierarchy through the
    Windows Time service.

The client configuration type `NT5DS` indicates that Windows Time
follows the Active Directory domain hierarchy. It does not mean that the
client should independently use the external NTP peer configured on the
domain controller.

**Important:** The PDC Emulator role should be verified before
describing DC01 as the forest-root PDC Emulator. The role check was
requested during this lab, but its output was not captured in the
available session record.

## 5. DC01 Time and Network Diagnosis

### 5.1 Check the Current Network Configuration

**VM:** DC01\
**Command:**

``` powershell
ipconfig
```

**Purpose:**

Display the network adapter configuration, including IPv4 addresses and
the default gateway.

**Why was it used?**

The external NTP peer could not initially be reached. Before changing
the Windows Time configuration, the network path needed to be checked. A
Host-only adapter provides private communication inside the lab, but it
does not automatically provide a route to the internet.

**Important observation:**

The Host-only adapter had no default gateway configured. This explained
why DC01 did not have an external route through that adapter.

### 5.2 Check the Current Windows Time Source

**VM:** DC01\
**Command:**

``` powershell
w32tm /query /source
```

**Purpose:**

Display the time source currently used by the Windows Time service.

**Why was it used?**

This identifies where Windows believes its current time is coming from.
During the initial investigation, DC01 was using its local hardware
clock rather than an external NTP peer.

**Observed result:**

`Local CMOS Clock`

**Explanation:**

`Local CMOS Clock` indicates that Windows is using the computer's local
hardware clock as its time source. It does not confirm synchronization
with an external NTP server.

### 5.3 Check the Windows Time Status

**VM:** DC01\
**Command:**

``` powershell
w32tm /query /status
```

**Purpose:**

Display the Windows Time service status, including information such as
the current source, stratum, last successful synchronization time, and
clock precision.

**Why was it used?**

The source alone does not provide enough information to assess
synchronization. The status output gives additional evidence about the
service's synchronization state.

**Result recording note:**

The command was run during the lab, but the complete output was not
preserved in the available session record. The exact stratum, last
successful synchronization time, and reference details should therefore
not be inferred.

## 6. Initial External NTP Connectivity Test

### 6.1 Test the External NTP Peer with W32Time

**VM:** DC01\
**Command:**

``` powershell
w32tm /stripchart /computer:time.cloudflare.com /samples:5 /dataonly
```

**Purpose:**

Measure the clock offset between DC01 and the specified time server by
sending a series of time requests.

**Why was it used?**

This command helps determine whether DC01 can communicate with the
external time source and estimates the difference between the local
clock and the remote source.

**Initial result:**

The command could not successfully obtain measurements before the NAT
adapter was added.

**Interpretation:**

The initial Host-only network configuration did not provide DC01 with an
external route. This was consistent with the missing default gateway
observed in the network configuration.

## 7. Troubleshooting External Connectivity

### 7.1 Add a NAT Adapter to DC01

**VM:** DC01 (VirtualBox configuration; VM was shut down before changing
the adapter configuration)

**Action:**

1.  Shut down DC01.
2.  Open the VirtualBox settings for DC01.
3.  Keep Adapter 1 configured as Host-only.
4.  Enable Adapter 2 and configure it as NAT.
5.  Start DC01.

**Purpose:**

Provide DC01 with outbound network connectivity while preserving the
private Host-only network used by the domain client.

**Observed result:**

The NAT adapter appeared in Windows as `Ethernet2` and received the IPv4
address:

``` text
10.0.3.15
```

**Why was this design used?**

The domain client still needs to communicate with DC01 over the private
Host-only network. Adding a separate NAT adapter gives DC01 an external
path without replacing the private lab network.

### 7.2 Verify DNS Resolution for the NTP Hostname

**VM:** DC01\
**Command:**

``` powershell
Resolve-DnsName time.cloudflare.com
```

**Purpose:**

Resolve the hostname `time.cloudflare.com` to its DNS records and IP
addresses.

**Why was it used?**

NTP configuration used a hostname rather than a fixed IP address. DNS
resolution needed to be checked separately from NTP communication.

**Observed result:**

The hostname resolved successfully.

**Interpretation:**

This confirmed that DC01 could resolve the external NTP hostname. DNS
resolution alone does not prove that NTP traffic is working; the NTP
request must also be tested.

### 7.3 Investigate TCP Port 123 Test Failure

**VM:** DC01\
**Command:**

``` powershell
Test-NetConnection time.cloudflare.com -Port 123
```

**Purpose:**

Test whether a TCP connection can be established to the specified host
and TCP port.

**Why was it used?**

The command was used while investigating connectivity to port 123.

**Observed result:**

The TCP connection test failed.

**Important technical distinction:**

NTP normally uses **UDP port 123**, not TCP port 123.
`Test-NetConnection -Port 123` tests TCP connectivity. Therefore, a
failed result from this command does not establish that NTP/UDP
communication is blocked.

This was an important troubleshooting distinction: a test must use the
same transport protocol as the service being investigated. A TCP test is
not a valid direct confirmation of NTP availability.

### 7.4 Re-test NTP with W32Time

**VM:** DC01\
**Command:**

``` powershell
w32tm /stripchart /computer:time.cloudflare.com /samples:5 /dataonly
```

**Purpose:**

Send time requests to the external NTP peer and display the measured
clock offsets.

**Why was it used?**

Unlike the TCP test, this command uses Windows Time functionality to
perform an NTP time measurement. It is therefore a more relevant test
for the NTP service.

**Observed result:**

The command returned a measurement of approximately:

``` text
+01.2793661
```

**Interpretation:**

DC01 was now able to obtain a time measurement from
`time.cloudflare.com`. This showed that the external NTP test was
working after the NAT adapter was added.

The measured offset is an estimate at the time of the request. It should
not be treated as a permanent offset or as proof by itself that the
Windows Time service has completed synchronization.

## 8. Configure DC01 to Use the External NTP Peer

### 8.1 Configure the Manual NTP Peer

**VM:** DC01\
**Command:**

``` powershell
w32tm /config /manualpeerlist:"time.cloudflare.com,0x8" /syncfromflags:manual /reliable:yes /update
```

**Purpose:**

Configure the Windows Time service to use `time.cloudflare.com` as a
manual time peer and update the service configuration.

**Why was it used?**

The external NTP peer had become reachable through the NAT adapter. The
next step was to configure Windows Time to use that peer rather than
relying on the local CMOS clock.

**Command breakdown:**

  ---------------------------------------------------------------------------------
  Parameter                                     Purpose
  --------------------------------------------- -----------------------------------
  `w32tm`                                       Windows Time service command-line
                                                utility

  `/config`                                     Changes Windows Time configuration

  `/manualpeerlist:"time.cloudflare.com,0x8"`   Specifies the manual time peer and
                                                peer flags

  `time.cloudflare.com`                         External NTP server hostname

  `0x8`                                         Configures the peer to use client
                                                mode

  `/syncfromflags:manual`                       Instructs Windows Time to use the
                                                manually configured peers

  `/reliable:yes`                               Marks this computer as a reliable
                                                time source for domain clients

  `/update`                                     Notifies the service that the
                                                configuration has changed
  ---------------------------------------------------------------------------------

**Observed result:**

The command returned a successful completion message.

**Important design note:**

`/reliable:yes` should be used only on the appropriate authoritative
domain time source, normally the PDC Emulator in the forest-root domain.
The PDC Emulator role should be verified before treating this setting as
a confirmed best-practice configuration for the domain. The command's
successful completion only confirms that Windows accepted the
configuration; it does not independently prove that DC01 is the correct
authoritative server.

### 8.2 Request Immediate Time Synchronization

**VM:** DC01\
**Command:**

``` powershell
w32tm /resync /rediscover
```

**Purpose:**

Ask the Windows Time service to synchronize immediately and rediscover
its time sources.

**Why was it used?**

Changing the peer configuration does not mean that the clock has already
synchronized. This command requests a synchronization operation using
the updated configuration.

**Command breakdown:**

  -----------------------------------------------------------------------
  Parameter                           Purpose
  ----------------------------------- -----------------------------------
  `w32tm`                             Windows Time service command-line
                                      utility

  `/resync`                           Requests immediate time
                                      synchronization

  `/rediscover`                       Rediscovers available time sources
                                      before synchronizing
  -----------------------------------------------------------------------

**Observed result:**

The command returned a successful completion message.

### 8.3 Verify the DC01 Time Source and Status

**VM:** DC01\
**Commands:**

``` powershell
w32tm /query /source
w32tm /query /status
```

**Purpose:**

-   `/query /source` displays the currently selected time source.
-   `/query /status` displays synchronization status and related
    metadata.

**Why were they used?**

A successful `/resync` message indicates that the request was accepted,
but verification should also inspect the source and status afterward.

**Result recording note:**

Both commands were run after the synchronization request, but their
complete output was not preserved in the available session record. The
exact final source, stratum, and last successful synchronization time
must be captured before documenting them as verified values.

### 8.4 Verify the PDC Emulator Role

**VM:** DC01\
**Command to run if the role has not yet been confirmed:**

``` powershell
Get-ADDomain | Select-Object PDCEmulator
```

**Purpose:**

Display the domain controller holding the PDC Emulator FSMO role for the
domain.

**Why is it important?**

The PDC Emulator in the forest-root domain normally acts as the
authoritative time source for the Active Directory domain hierarchy.
Confirming the role helps establish whether DC01 is the appropriate
server to configure as the reliable domain time source.

**Result recording note:**

This check was requested during the lab, but its output was not captured
in the available session record. Do not record DC01 as the confirmed PDC
Emulator until the command output has been checked.

## 9. Synchronize CLIENT01-NEW Through the Domain

### 9.1 Understand the Client Time Configuration

**VM:** CLIENT01-NEW\
**Command:**

``` powershell
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\W32Time\Parameters" | Select-Object Type, NtpServer
```

**Purpose:**

Read the Windows Time service parameters from the registry, specifically
the time synchronization type and configured NTP server value.

**Why was it used?**

The client needed to be checked to understand whether it was configured
to use the Active Directory domain hierarchy or a manually configured
external peer.

**Observed result:**

``` text
Type      : NT5DS
NtpServer : time.windows.com,0x9
```

**Interpretation:**

`NT5DS` means the Windows Time service follows the Active Directory
domain time hierarchy.

The presence of `time.windows.com,0x9` in the `NtpServer` registry value
does not mean that the client is necessarily using that external peer
directly. With `Type` set to `NT5DS`, the domain hierarchy is the
relevant synchronization mechanism.

### 9.2 Check the Client's Current Time Source

**VM:** CLIENT01-NEW\
**Command:**

``` powershell
w32tm /query /source
```

**Purpose:**

Display the time source currently used by the client.

**Observed result:**

``` text
dc01.corp.test
```

**Interpretation:**

The client was using DC01 as its time source, consistent with the
expected Active Directory domain time hierarchy.

### 9.3 Review Windows Time Service Events

**VM:** CLIENT01-NEW\
**Command:**

``` powershell
Get-WinEvent -FilterHashtable @{ LogName = 'System'; ProviderName = 'Microsoft-Windows-Time-Service' } -MaxEvents 10 | Format-List TimeCreated, Id, Message
```

**Purpose:**

Retrieve recent Windows Time service events from the System event log.

**Why was it used?**

When time synchronization fails or the local clock differs significantly
from the domain controller, the event log can provide diagnostic
evidence. It helps identify the reported offset and the Windows Time
service's response.

**Command breakdown:**

  ---------------------------------------------------------------------------------------
  Element                                             Purpose
  --------------------------------------------------- -----------------------------------
  `Get-WinEvent`                                      Retrieves event log records

  `-FilterHashtable`                                  Applies structured event-log
                                                      filters

  `LogName = 'System'`                                Searches the System log

  `ProviderName = 'Microsoft-Windows-Time-Service'`   Filters for Windows Time service
                                                      events

  `-MaxEvents 10`                                     Returns up to the latest 10
                                                      matching events

  `Format-List`                                       Displays selected event properties
                                                      in a readable list

  `TimeCreated`                                       Event timestamp

  `Id`                                                Event ID

  `Message`                                           Full event message
  ---------------------------------------------------------------------------------------

**Observed events:**

-   Event ID 52 reported a clock offset of approximately 60,824 seconds.
-   Event ID 50 reported a time difference greater than 5,000
    milliseconds for 900 seconds.

**Interpretation:**

The event records showed that the client had experienced a very large
time difference. An offset of 60,824 seconds is approximately 16 hours,
53 minutes, and 44 seconds.

The event records are evidence of the reported time difference. They do
not, by themselves, prove that the time-zone setting was the root cause.
The actual UTC clock values and the synchronization source must also be
considered.

### 9.4 Request Client Synchronization

**VM:** CLIENT01-NEW\
**Command:**

``` powershell
w32tm /resync /rediscover
```

**Purpose:**

Ask the client Windows Time service to rediscover its domain time source
and synchronize.

**Why was it used?**

After DC01's external time configuration had been updated, the client
needed to synchronize against the domain time hierarchy.

**Observed result:**

The command returned a successful completion message.

### 9.5 Verify Client Time Status

**VM:** CLIENT01-NEW\
**Commands:**

``` powershell
w32tm /query /source
w32tm /query /status
```

**Purpose:**

Confirm the selected time source and inspect the synchronization status.

**Result recording note:**

The commands were run after the client synchronization request. The
source had previously been observed as `dc01.corp.test`. The complete
post-synchronization status output was not preserved, so the final
stratum and last successful synchronization timestamp should be captured
before being reported as verified.

## 10. Final Time Difference Verification

### 10.1 Measure the Difference Between CLIENT01-NEW and DC01

**VM:** CLIENT01-NEW\
**Command:**

``` powershell
w32tm /stripchart /computer:dc01.corp.test /samples:5 /dataonly
```

**Purpose:**

Measure the time difference between CLIENT01-NEW and the domain
controller.

**Why was it used?**

The client should synchronize through DC01. Measuring the offset
directly provides a practical check of how closely the two systems'
clocks agree.

**Observed result:**

The measurements were approximately:

``` text
-01.8208272s
-02s
-03s
-03s
```

**Interpretation:**

The measured difference was approximately 1--3 seconds. This is
substantially below the typical five-minute Kerberos maximum clock-skew
tolerance.

This result demonstrates that the client and domain controller clocks
were closely aligned at the time of measurement. It does not replace
checking the Windows Time service status and source on both machines.

## 11. Troubleshooting Summary

  --------------------------------------------------------------------------------------------------
  Symptom                 Diagnostic action                                  Finding / resolution
  ----------------------- -------------------------------------------------- -----------------------
  DC01 used               `w32tm /query /source`                             DC01 was not yet using
  `Local CMOS Clock`                                                         the external NTP peer

  External NTP            `w32tm /stripchart ...` and `ipconfig`             Host-only adapter had
  measurement initially                                                      no default gateway
  failed                                                                     

  TCP port 123 test       `Test-NetConnection ... -Port 123`                 This tested TCP,
  failed                                                                     whereas NTP normally
                                                                             uses UDP

  External hostname       `Resolve-DnsName time.cloudflare.com`              DNS resolution
  needed validation                                                          succeeded after NAT was
                                                                             added

  External NTP            `w32tm /stripchart ...`                            A measurement of
  measurement succeeded                                                      approximately +1.279
  after network change                                                       seconds was returned

  Client reported a very  `Get-WinEvent` for Windows Time events             Event ID 52 reported
  large time difference                                                      approximately 60,824
                                                                             seconds

  Client needed to update `w32tm /resync /rediscover`                        Synchronization request
  its time                                                                   completed successfully

  Final client/DC offset  `w32tm /stripchart /computer:dc01.corp.test ...`   Final measured
  needed confirmation                                                        difference was
                                                                             approximately 1--3
                                                                             seconds
  --------------------------------------------------------------------------------------------------

## 12. Key Technical Questions and Explanations

### 12.1 Does a failed TCP port 123 test mean NTP is blocked?

No. NTP normally uses UDP port 123. `Test-NetConnection -Port 123`
checks TCP. A failed TCP connection does not prove that UDP-based NTP
communication has failed. Use an NTP-aware test such as
`w32tm /stripchart` to test the time service.

### 12.2 Why did DC01 need a NAT adapter?

The Host-only adapter was intended for private communication inside the
lab. It did not provide an external route. The NAT adapter provided
outbound connectivity to the external NTP service while leaving the
private domain network in place.

### 12.3 Why does the client use DC01 instead of Cloudflare directly?

The client was configured with `Type = NT5DS`, which means it follows
the Active Directory domain time hierarchy. In a domain environment,
clients normally synchronize with domain controllers rather than
independently selecting an external public NTP server.

### 12.4 Does changing the time zone synchronize the clock?

No. A time zone controls how UTC time is displayed as local time.
Windows maintains system time internally in UTC and converts it for
local display. Changing the time zone does not itself synchronize the
clock with an NTP server.

A time-zone mismatch can make displayed times look different, but it is
not the same as a real UTC clock offset. Kerberos clock-skew problems
concern the underlying time difference, not simply different time-zone
labels.

### 12.5 Does a successful `/resync` message prove synchronization is correct?

Not by itself. It indicates that the synchronization request completed,
but the source and status should also be checked with:

``` powershell
w32tm /query /source
w32tm /query /status
```

For an additional offset measurement, use:

``` powershell
w32tm /stripchart /computer:dc01.corp.test /samples:5 /dataonly
```

## 13. Final Verification Results

  Verification item                         Result
  ----------------------------------------- --------------------------------
  DC01 NAT adapter                          Received `10.0.3.15`
  DNS resolution of external NTP hostname   Successful
  External NTP stripchart test              Successful after NAT was added
  DC01 manual peer configuration            Command completed successfully
  DC01 resynchronization request            Command completed successfully
  Client time source                        `dc01.corp.test` observed
  Client time configuration type            `NT5DS`
  Client resynchronization request          Command completed successfully
  Client-to-DC measured offset              Approximately 1--3 seconds
  DC01 final source and status output       Exact output not preserved
  Client final status output                Exact output not preserved
  PDC Emulator role verification            Output not preserved

## 14. Outcome and Next Stage

The lab successfully established an external NTP measurement path from
DC01 after adding a NAT adapter. DC01 was configured to use
`time.cloudflare.com`, and the client was confirmed to use
`dc01.corp.test` as its domain time source. The final measured
difference between CLIENT01-NEW and DC01 was approximately 1--3 seconds.

This is a useful foundation for the next Kerberos investigation.
Kerberos authentication relies on accurate timestamps, so time
synchronization should be understood and verified before analyzing the
authentication exchange in Wireshark.

Before treating the time configuration as fully documented, capture the
missing final outputs for:

**DC01:**

``` powershell
Get-ADDomain | Select-Object PDCEmulator
w32tm /query /source
w32tm /query /status
```

**CLIENT01-NEW:**

``` powershell
w32tm /query /source
w32tm /query /status
```

These outputs will confirm the authoritative domain time role, selected
time sources, stratum, and last successful synchronization timestamps.
