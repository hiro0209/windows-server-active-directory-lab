# Windows System Resource Monitoring

## 1. Overview

This lab documents a Windows performance investigation on a Windows 11 client VM, `CLIENT01-NEW`.

The scenario is a Help Desk request:

"My PC feels slow."

The objective is to investigate CPU, Memory, Disk, and Network activity using Task Manager, Resource Monitor, and PowerShell.

The purpose is to identify potential resource bottlenecks, understand Windows performance metrics, and interpret diagnostic results.

---

## 2. Lab Environment

| Item | Value |
|---|---|
| Client VM | CLIENT01-NEW |
| Operating System | Windows 11 |
| Domain | corp.test |
| RAM | 4.0 GB |
| Host-only IP | 192.168.56.105 |
| Virtualisation | VirtualBox |

---

## 3. Investigation Workflow

The following steps were performed:

1. Inspect CPU and Memory using Task Manager.
2. Open Resource Monitor using `resmon`.
3. Investigate detailed Memory usage.
4. Investigate Disk Activity, Response Time, and Disk Queue Length.
5. Investigate Network Activity and TCP Connections.
6. Inspect Listening Ports and Firewall Status.
7. Use PowerShell to inspect TCP connection states.
8. Evaluate the observations and document the findings.

---

## 4. Task Manager Investigation

### 4.1 CPU Usage

**Location:** Task Manager → Processes

The Processes tab was inspected, and processes were sorted by CPU usage.

**Observed results:**

- Microsoft Edge was the process with the highest CPU usage.
- CPU usage was approximately 1.9%.
- Edge memory usage was approximately 110 MB.

**Interpretation:**

Microsoft Edge was the highest CPU-consuming process at the time of observation.

However, 1.9% CPU usage is relatively low. Being the highest process in the list does not automatically mean that it is causing a performance problem.

A process should be monitored over time before determining whether it is responsible for high CPU usage.

### 4.2 Memory Usage

**Location:** Task Manager → Performance → Memory

**Observed results:**

| Metric | Value |
|---|---:|
| Total RAM | 4.0 GB |
| In Use (Compressed) | 2.7 GB |
| Compressed | 312 MB |
| Available | 1.3 GB |

Approximately 67.5% of the displayed RAM was in use.

**Interpretation:**

The system was using approximately 2.7 GB out of 4.0 GB.

However, approximately 1.3 GB was still available.

This observation alone does not indicate memory exhaustion.

### 4.3 What is RAM?

RAM stands for Random Access Memory.

It is the computer's temporary working memory.

Applications and the operating system use RAM to store data and instructions that need to be accessed quickly.

Unlike permanent storage such as an SSD, RAM is volatile memory. Its contents are generally lost when the computer is powered off.

### 4.4 What is Memory Compression?

Windows uses Memory Compression to reduce the amount of physical memory required by applications and the operating system.

When memory pressure increases, Windows can compress memory pages and keep the compressed data in RAM instead of immediately moving those pages to the page file on disk.

This can reduce disk paging and improve system responsiveness.

In this lab:

- Memory Compression was approximately 312 MB in Task Manager.
- This value alone does not indicate a problem.

<img width="955" height="724" alt="image" src="https://github.com/user-attachments/assets/c7eab42e-3ca6-497e-9140-9516f97b8a11" />


<img width="1273" height="1140" alt="image" src="https://github.com/user-attachments/assets/c4513269-c44b-41e8-9ec3-dc68e4ed7f8b" />



<img width="1594" height="1204" alt="image" src="https://github.com/user-attachments/assets/5ba456e0-9b3c-4974-bd77-bfa5b00e8384" />


---

## 5. Resource Monitor

### 5.1 What is Resource Monitor?

Resource Monitor is a built-in Windows diagnostic tool.

It provides more detailed information about CPU, Memory, Disk, and Network activity than the basic Processes view in Task Manager.

**How to open Resource Monitor:**

1. Press `Win + R`.
2. Enter `resmon`.
3. Press Enter.

<img width="499" height="259" alt="image" src="https://github.com/user-attachments/assets/17b7c91f-a532-4f63-9b49-c762a739884c" />

<img width="568" height="759" alt="image" src="https://github.com/user-attachments/assets/ec374e36-d803-45fb-af76-e36931bf76bb" />

### 5.2 Task Manager vs Resource Monitor

| Tool | Main Purpose |
|---|---|
| Task Manager | Quick overview of system performance and running applications |
| Resource Monitor | More detailed investigation of individual processes and resource activity |

Resource Monitor has four main tabs:

| Tab | Purpose |
|---|---|
| CPU | Investigate CPU usage and process activity |
| Memory | Investigate process memory usage and physical memory |
| Disk | Investigate disk I/O, response time, and queue length |
| Network | Investigate network traffic, TCP connections, and listening ports |

---

## 6. Memory Investigation in Resource Monitor

### 6.1 Physical Memory

**Location:** Resource Monitor → Memory

The Physical Memory section displayed the following values:

| Metric | Observed Value |
|---|---:|
| In Use | 2,837 MB |
| Modified | 37 MB |
| Standby | 1,189 MB |
| Free | 16 MB |

The values total approximately 4,079 MB.

Small differences between Task Manager and Resource Monitor can occur because the tools use different memory categories and update at different times.

<img width="1597" height="664" alt="image" src="https://github.com/user-attachments/assets/43194ff5-bbcd-4937-b742-b6730c60411b" />


### 6.2 Meaning of Physical Memory Categories

**In Use**

Physical memory currently being used by Windows, applications, drivers, and other processes.

**Modified**

Memory pages that have been changed and may need to be written to their backing storage before they can be reused.

**Standby**

Memory containing cached data that can generally be reclaimed quickly when applications need more memory.

**Free**

Physical memory that is currently not being used to store active or cached data.




### 6.3 Why Can Free Memory Be Very Low?

During this lab, Free memory was only 16 MB.

Initially, this may look like a memory shortage.

However, Windows uses available RAM to cache frequently accessed data.

Standby memory can be reclaimed when applications require additional memory.

Therefore, low Free memory does not automatically mean that the system is running out of RAM.

It is important to evaluate Available memory, In Use memory, Standby memory, and system behaviour together.

### 6.4 Working Set

Working Set refers to the memory pages belonging to a process that are currently resident in physical RAM.

In simple terms, it represents how much of a process's memory is currently held in RAM.

The Resource Monitor Processes list was sorted by Working Set.

**Observed results:**

| Process | PID | Working Set |
|---|---:|---:|
| Memory Compression | 2620 | 327,600 KB |
| explorer.exe | 5252 | 175,560 KB |

Memory Compression was the largest entry at approximately 320 MB.

`explorer.exe` was approximately 171 MB.

### 6.5 What is Memory Compression Process?

Memory Compression is a Windows memory-management feature.

It compresses memory pages and keeps them in RAM.

This helps Windows reduce the need to move memory pages to the page file on disk.

A relatively large Memory Compression Working Set does not automatically indicate a fault.

It should be evaluated together with overall memory pressure and system responsiveness.

### 6.6 What is explorer.exe?

`explorer.exe` is the Windows shell process.

It manages important Windows user-interface components, including:

- Desktop
- Taskbar
- File Explorer
- Start-menu-related shell interactions

It is a normal Windows process.

### 6.7 What is Private Working Set?

Private Working Set represents the portion of a process's resident Working Set that is private to that process rather than shared with other processes.

It is important not to confuse Private Working Set with Private Bytes or total committed memory.

These are different memory measurements.

---

## 7. Disk Investigation

**Location:** Resource Monitor → Disk

The Disk tab was inspected to investigate disk activity and potential storage bottlenecks.

Two important metrics were examined:

- Response Time
- Disk Queue Length

### 7.1 What is Response Time?

Response Time represents how long a disk I/O request takes to complete.

It is displayed in milliseconds (ms).

A higher Response Time means that the disk operation took longer to respond.

For example:

| Response Time | General Interpretation |
|---|---|
| 1 ms | Very short response time |
| 3 ms | Short response time |
| 50 ms | Longer response time |
| 200 ms | Substantially longer response time |

The interpretation depends on the storage device, workload, and observation period.

A single high value does not automatically indicate a disk failure.

Sustained high response times may indicate storage performance problems.

**Observed results:**

- 3 ms
- 3 ms
- 3 ms
- 2 ms
- 1 ms

The highest observed Response Time was approximately 3 ms.

**Interpretation:**

The observed disk response times were low during this investigation.

There was no clear evidence of a disk response-time bottleneck in this sample.

### 7.2 What is Disk Queue Length?

Disk Queue Length represents disk I/O requests waiting to be processed.

When applications request data from storage, those requests may need to wait in a queue.

If requests continuously accumulate faster than the storage device can process them, the queue may increase.

A sustained high queue can indicate that the storage system is struggling to keep up with demand.

**Observed results:**

- Approximately 0.01
- Approximately 0.02
- Maximum observed value: approximately 0.02

An earlier observation was approximately 0.021.

**Interpretation:**

The observed queue length was very low.

There was no clear evidence of a disk I/O queue bottleneck during this sample.

However, disk performance should be monitored over time because temporary spikes may not appear in a short observation.


<img width="1017" height="768" alt="image" src="https://github.com/user-attachments/assets/5346be1d-416e-4b75-901a-785d1f868a41" />


---

## 8. Network Investigation

**Location:** Resource Monitor → Network

The Network tab was used to investigate network activity, TCP connections, and listening ports.

The following sections were inspected:

1. Processes with Network Activity
2. Network Activity
3. TCP Connections
4. Listening Ports

### 8.1 Network Activity

Network Activity displays how much data individual processes send and receive.

Important columns include:

| Column | Meaning |
|---|---|
| Send (B/sec) | Amount of data sent per second |
| Receive (B/sec) | Amount of data received per second |
| Total (B/sec) | Combined sending and receiving rate |

**Observed values:**

Send:

- 95 B/sec
- 14 B/sec
- 21 B/sec

Receive:

- 178 B/sec
- 89 B/sec
- 17 B/sec
- 34 B/sec

Total values were also observed on the screen:

- 178 B/sec
- 68 B/sec
- 20 B/sec
- 9 B/sec

The exact row-to-process mapping was not preserved in the notes.

Therefore, these values are recorded as approximate observations and are not attributed to individual processes.


<img width="803" height="260" alt="image" src="https://github.com/user-attachments/assets/ed23beb0-aade-49d7-8a3c-20f847a60bab" />


**Interpretation:**

The observed network traffic was relatively low at the time of inspection.

However, low traffic does not automatically mean that the network connection is healthy.

Other factors such as latency, packet loss, DNS resolution, routing, and connectivity should also be investigated when necessary.

### 8.2 TCP Connections

TCP Connections displays information about active TCP endpoints and their remote connection details.

The following information was observed:

| Process | PID | Remote Address | Remote Port |
|---|---:|---|---:|
| svchost.exe (netsvcs -p) | 3432 | 198.18.0.7 | 443 |
| msedgewebview2.exe | 460 | 198.18.0.120 | 443 |
| StartMenuExperienceHost.exe | 6828 | 198.18.0.66 | 443 |

Additional observations:

- `msedgewebview2.exe` appeared three times.
- `StartMenuExperienceHost.exe` appeared twice.
- `svchost.exe` appeared once.

### 8.3 What is svchost.exe?

`svchost.exe` stands for Service Host.

It is a Windows process used to host Windows services.

Multiple instances of `svchost.exe` can run simultaneously because different Windows services may run in separate service-host processes.

The presence of multiple instances is normal.

### 8.4 What is msedgewebview2.exe?

`msedgewebview2.exe` belongs to Microsoft Edge WebView2 Runtime.

WebView2 allows Windows applications to display web-based content using Microsoft Edge technology.

It is commonly used by applications that integrate web interfaces.

Multiple instances may exist because the runtime can use separate processes for different components.

### 8.5 What is StartMenuExperienceHost.exe?

`StartMenuExperienceHost.exe` is a Windows component responsible for the Start menu experience.

It is part of the Windows user interface.

Multiple entries can appear in diagnostic views depending on process activity and connections.

### 8.6 What is Port 443?

Port 443 is commonly used for HTTPS communication.

HTTPS encrypts communication between clients and servers using TLS.

However, seeing port 443 does not automatically prove that the traffic is HTTPS.

The actual application protocol should be verified if deeper investigation is required.

### 8.7 What is 198.18.0.0/15?

The observed remote addresses included:

- 198.18.0.7
- 198.18.0.120
- 198.18.0.66

These addresses belong to the `198.18.0.0/15` range.

This range is reserved for benchmarking and performance testing.

It is not an ordinary public Internet address range.

The exact reason these addresses appeared in the VM environment was not established during this investigation.

### 8.8 Why Was There No State Column in Resource Monitor?

The Resource Monitor TCP Connections table does not provide the same TCP State column as PowerShell's `Get-NetTCPConnection`.

This was an important distinction during the investigation.

PowerShell provides a more direct way to inspect TCP connection states.


<img width="800" height="254" alt="image" src="https://github.com/user-attachments/assets/c61e827e-ba1b-4411-9a17-63a78ffa910a" />


---

## 9. Listening Ports Investigation

**Location:** Resource Monitor → Network → Listening Ports

Listening Ports displays processes that have opened local ports and are waiting for incoming connection requests.

The following columns were inspected:

| Column | Meaning |
|---|---|
| Image | Process name |
| PID | Process ID |
| Address | Local IP address used for listening |
| Port | Local port number |
| Protocol | TCP or UDP |
| Firewall Status | Windows Firewall status shown by Resource Monitor |

### 9.1 Observed Listening Ports

The following entries were observed:

| Process | PID | Address | Port | Protocol | Firewall Status |
|---|---:|---|---:|---|---|
| svchost.exe (LocalService) | 1240 | IPv6 unspecified | 123 | UDP | Allowed, not restricted |
| svchost.exe (RPCSS -p) | 468 | IPv4 unspecified | 135 | TCP | Allowed, not restricted |
| System | 4 | 192.168.56.105 | 137 | UDP | Allowed, not restricted |
| svchost.exe (LocalServiceAndNoImpersonationRedirectionGuard -p) | 8456 | fe80::aa66:772:aa4b:f74f | 1900 | UDP | Allowed, not restricted |

### 9.2 Port 123 - NTP

Port 123 uses UDP and is commonly associated with Network Time Protocol (NTP).

NTP synchronizes system clocks with time sources across a network.

Accurate time synchronization is important for:

- Kerberos authentication
- Domain environments
- Event log correlation
- Security investigations

### 9.3 Port 135 - RPC

Port 135 uses TCP and is associated with the Microsoft RPC Endpoint Mapper.

RPC stands for Remote Procedure Call.

Windows uses RPC for communication between processes and services.

The Endpoint Mapper helps clients identify the appropriate endpoint for an RPC service.

### 9.4 Port 137 - NetBIOS Name Service

Port 137 uses UDP and is associated with NetBIOS Name Service (NBNS).

It is used for NetBIOS name resolution on networks that support NetBIOS over TCP/IP.

It should not be confused with DNS.

DNS primarily resolves domain names, while NetBIOS name resolution is a separate legacy naming mechanism.

### 9.5 Port 1900 - SSDP

Port 1900 uses UDP and is commonly associated with Simple Service Discovery Protocol (SSDP).

SSDP is used by devices and services to discover each other on a local network.

It is commonly associated with Universal Plug and Play (UPnP).

### 9.6 What Does IPv4 Unspecified Mean?

An IPv4 unspecified listening address commonly appears as:

`0.0.0.0`

It means that the socket is bound to all applicable local IPv4 interfaces.

It does not mean that the computer has no IP address.

### 9.7 What Does IPv6 Unspecified Mean?

An IPv6 unspecified listening address commonly appears as:

`::`

It means that the socket is bound to all applicable local IPv6 interfaces.

It is the IPv6 equivalent of the IPv4 wildcard address `0.0.0.0`.

### 9.8 What Does fe80:: Mean?

The address:

`fe80::aa66:772:aa4b:f74f`

belongs to the IPv6 link-local address range.

Link-local addresses are used for communication on the local network link.

They are not normally routed across the public Internet.

### 9.9 What Does Allowed, Not Restricted Mean?

Resource Monitor displays Firewall Status information for listening ports.

In this investigation, the observed entries showed:

`Allowed, not restricted`

This does not guarantee that an external device can connect successfully.

Actual reachability also depends on:

- Firewall rules
- Network profile
- Listening address
- Routing
- Network isolation
- Other security controls

A listening port should not automatically be considered a security vulnerability.

Its purpose, exposure, and necessity must be evaluated.


<img width="1597" height="1204" alt="image" src="https://github.com/user-attachments/assets/3a5a85d1-f4f9-4570-ba72-ef9eb576c226" />


---

## 10. PowerShell TCP Connection Investigation

### 10.1 Get-NetTCPConnection

PowerShell provides a way to inspect TCP endpoints and their states.

The following command was executed on `CLIENT01-NEW`.

**Command:**

```powershell
Get-NetTCPConnection | Select-Object -First 10 LocalAddress, LocalPort, RemoteAddress, RemotePort, State
```

<img width="1588" height="1134" alt="image" src="https://github.com/user-attachments/assets/413dc8cf-a4b6-4a4d-9915-e01df7d06fb8" />


<img width="1579" height="534" alt="image" src="https://github.com/user-attachments/assets/55fe47b5-6e82-4e70-bc15-41e420cea974" />


**Command explanation:**

| Part | Purpose |
|---|---|
| `Get-NetTCPConnection` | Retrieves TCP connection information |
| `Select-Object` | Selects specific properties to display |
| `-First 10` | Displays only the first 10 objects |
| `LocalAddress` | Local IP address |
| `LocalPort` | Local TCP port |
| `RemoteAddress` | Remote IP address |
| `RemotePort` | Remote TCP port |
| `State` | Current TCP connection state |

**Observed result:**

All 10 displayed entries showed:

`Listen`

### 10.2 What Does Listen Mean?

`Listen` is a TCP state.

It means that a local TCP endpoint is waiting for an incoming connection request.

It does not mean that an established connection currently exists.

For example:

- `Listen`: Waiting for incoming TCP connections.
- `Established`: A TCP connection has been established.
- `TimeWait`: A connection has been closed, but the endpoint remains in this state temporarily.
- `CloseWait`: The remote endpoint has closed its connection, and the local application has not yet closed its side.

### 10.3 Important Limitation of the Command

The command used `-First 10`.

Therefore, it only displayed the first 10 returned entries.

The result does not prove that every TCP endpoint on the computer was in the `Listen` state.

It only confirms that the 10 displayed entries showed `Listen`.

### 10.4 Optional: Count TCP States

The following command can be used in a future investigation to group TCP connections by state.

```powershell
Get-NetTCPConnection |
    Group-Object -Property State |
    Select-Object Name, Count
```

**Purpose:**

- `Group-Object -Property State` groups TCP endpoints by their state.
- `Select-Object Name, Count` displays each state and the number of entries.

This command was not executed during this lab.

It is included as an optional diagnostic command for future use.

---

## 11. Key Technical Concepts Learned

| Concept | Explanation |
|---|---|
| RAM | Temporary working memory used by Windows and applications |
| Memory Compression | Compresses memory pages to reduce physical memory usage |
| Working Set | Process memory currently resident in physical RAM |
| Private Working Set | Resident process memory that is private rather than shared |
| Standby Memory | Cached memory that can generally be reclaimed when needed |
| Modified Memory | Changed memory pages that may need to be written to backing storage |
| Response Time | Time taken for a disk I/O request to complete |
| Disk Queue Length | Number of disk I/O requests waiting to be processed |
| Network Activity | Amount of data sent and received by processes |
| TCP Connection | A communication endpoint using the TCP protocol |
| Listening Port | A local TCP endpoint waiting for incoming connections |
| NTP | Network Time Protocol, used for clock synchronization |
| RPC | Remote Procedure Call, used for communication between processes and services |
| NetBIOS | A legacy network naming and communication interface |
| SSDP | Simple Service Discovery Protocol, used for local device discovery |
| IPv4 Unspecified | `0.0.0.0`, a wildcard address for applicable IPv4 interfaces |
| IPv6 Unspecified | `::`, a wildcard address for applicable IPv6 interfaces |
| IPv6 Link-Local | An IPv6 address used on the local network link |
| TCP Listen | A TCP endpoint waiting for an incoming connection |
| TCP Established | A TCP connection that has been established |

---

## 12. Overall Findings

The investigation produced the following observations.

| Resource | Findings | Assessment |
|---|---|---|
| CPU | Microsoft Edge was highest at approximately 1.9% | No clear CPU bottleneck observed |
| Memory | 2.7 GB in use, 1.3 GB available | No clear memory exhaustion observed |
| Disk | Response time approximately 1–3 ms | No clear disk response-time bottleneck observed |
| Disk Queue | Approximately 0.01–0.02 | Very low observed queue length |
| Network | Relatively low traffic during observation | No conclusion about overall network health |
| TCP | First 10 PowerShell entries showed Listen | Only the displayed entries were verified |
| Listening Ports | Four entries inspected | Port purpose and exposure should be assessed in context |

### 13. Conclusion

The investigation did not identify a clear CPU, Memory, or Disk bottleneck during the observation period.

Network traffic was relatively low, and several listening ports were identified and investigated.

However, this was a short diagnostic session in a virtualised environment.

Further investigation would be necessary if a user continued to experience slow performance.

Possible next steps include:

- Monitoring performance over a longer period.
- Checking Windows Event Viewer for relevant errors.
- Investigating application-specific performance issues.
- Checking network latency and packet loss.
- Checking host machine resource usage because the client is running in VirtualBox.

---
