# Kerberos Authentication Analysis with Wireshark

## 1. Overview

This lab investigates the Kerberos authentication process in a Windows Server 2025 Active Directory environment.

The objective was to capture and analyse actual network traffic generated during Kerberos authentication and understand how a Windows client obtains a Ticket-Granting Ticket (TGT), requests a Service Ticket, and authenticates to an LDAP service.

Wireshark was used to capture and inspect network packets. PowerShell commands were used to trigger Kerberos authentication and LDAP directory queries.

The investigation focused on three main stages:

1. **AS Exchange:** Obtaining a Ticket-Granting Ticket (TGT).
2. **TGS Exchange:** Obtaining a Service Ticket for LDAP.
3. **AP Exchange:** Presenting the Service Ticket to the LDAP service and performing authentication.

## 2. Lab Environment

| Component | Configuration |
|---|---|
| Domain Controller | Windows Server 2025 |
| Client | Windows 11 |
| Domain Name | `corp.test` |
| NetBIOS Name | `CORP` |
| Domain Controller IP | `192.168.56.10` |
| Client IP | `192.168.56.105` |
| Virtualisation | Oracle VirtualBox |
| Packet Analysis | Wireshark |
| Directory Protocol | LDAP |
| Authentication Protocol | Kerberos V5 |

The client and domain controller communicated through the VirtualBox Host-only network.

Wireshark was running on the client VM and capturing traffic on the Host-only Ethernet interface.

## 3. Investigation Methodology

The investigation followed this sequence:

1. Prepare packet capture on the client VM.
2. Clear the existing Kerberos ticket cache.
3. Trigger Kerberos authentication.
4. Capture and inspect AS Exchange packets.
5. Request an LDAP Service Ticket.
6. Capture and inspect TGS Exchange packets.
7. Perform an LDAP directory query.
8. Inspect the LDAP SASL authentication exchange.
9. Identify the AP-REQ and AP-REP tokens.
10. Verify the LDAP Bind result.

## 4. AS Exchange – Obtaining a TGT

### 4.1 Clearing the Kerberos Ticket Cache

**Execution location:** CLIENT01-NEW – PowerShell

```powershell
klist purge
```

**Purpose:**

This command removes the currently cached Kerberos tickets for the current logon session.

Clearing the cache helps observe a new authentication sequence rather than relying on previously cached tickets.

**Observed result:**

The existing Kerberos tickets were purged. A new authentication sequence could then be observed.

Before:
<img width="1200" height="978" alt="image" src="https://github.com/user-attachments/assets/88b242be-87e5-40c3-8076-86e716eea184" />



After:

<img width="499" height="165" alt="image" src="https://github.com/user-attachments/assets/bf69634b-312c-4557-811e-a91afd3f5698" />


### 4.2 Capturing the AS Exchange

The following packet sequence was observed in Wireshark:

| Order | Packet | Description |
|---|---|---|
| 1 | AS-REQ | The client requests a TGT. |
| 2 | KRB-ERROR: PREAUTH_REQUIRED | The KDC indicates that pre-authentication is required. |
| 3 | AS-REQ | The client sends another request containing pre-authentication data. |
| 4 | AS-REP | The KDC responds with the TGT and session key information. |

### 4.3 Understanding Pre-authentication

The first KDC response contained:

```text
KDC_ERR_PREAUTH_REQUIRED
Error Code: 25
```

This response indicates that the KDC requires pre-authentication before issuing a TGT. In this observed sequence, it was part of the normal authentication process rather than evidence of a failed login.

<img width="1075" height="817" alt="image" src="https://github.com/user-attachments/assets/cfbcbcfd-9192-40fe-912b-8c1c548088a6" />



The second AS-REQ contained:

```text
PA-ENC-TIMESTAMP
PA-DATA Type: 2
```

This pre-authentication data contains an encrypted timestamp. It allows the KDC to verify that the client possesses the appropriate long-term key without sending the user's password in plaintext.

<img width="1119" height="844" alt="image" src="https://github.com/user-attachments/assets/80320850-eb8e-4f99-b3da-ef307e9ce5c1" />


### 4.4 Inspecting the AS-REP

The AS-REP packet contained a ticket with the service principal:

```text
krbtgt/CORP.TEST
```

This identifies the ticket as a Ticket-Granting Ticket (TGT).

The ticket encryption type observed was:

```text
AES256-CTS-HMAC-SHA1-96
Encryption Type: 18
```

The TGT is encrypted using the `krbtgt` account's key. The encrypted data in the outer AS-REP response is separately protected using the client's long-term key.

<img width="1117" height="870" alt="image" src="https://github.com/user-attachments/assets/38e518ab-d4cb-410a-b8fe-ab96a79ecb8b" />


## 5. TGS Exchange – Requesting an LDAP Service Ticket

### 5.1 Requesting the LDAP Service Ticket

**Execution location:** CLIENT01-NEW – PowerShell

```powershell
klist get ldap/dc01.corp.test
```

**Purpose:**

This command requests a Kerberos Service Ticket for the LDAP service hosted by the domain controller.

**Parameter explanation:**

- `klist`: Windows utility for managing and displaying Kerberos tickets.
- `get`: Requests a ticket for a specified service principal.
- `ldap/dc01.corp.test`: The LDAP Service Principal Name (SPN).

### 5.2 Observed TGS Exchange

The following packets were observed:

| Packet | Description |
|---|---|
| TGS-REQ | The client requests an LDAP Service Ticket. |
| TGS-REP | The KDC issues the requested Service Ticket. |

The TGS-REQ contained the following service name:

```text
ldap/dc01.corp.test
```

This identifies the requested LDAP service.

<img width="1180" height="702" alt="image" src="https://github.com/user-attachments/assets/530a689c-9b72-4aeb-82cc-f06a13bddc23" />



### 5.3 Inspecting PA-TGS-REQ

The TGS-REQ contained `PA-TGS-REQ` data.

Inside this data, an AP-REQ token was present. Its ticket referenced:

```text
krbtgt/CORP.TEST
```

This is the client's presentation of its existing TGT to the KDC as part of the TGS request.

**Important distinction:**

The AP-REQ inside PA-TGS-REQ is used to authenticate the client's request to the KDC. It is not the AP-REQ subsequently presented to the LDAP service.

<img width="1128" height="814" alt="image" src="https://github.com/user-attachments/assets/9a389dd4-fcc4-4d39-9c1c-f37a9e1a26e9" />


### 5.4 Inspecting the TGS-REP

The TGS-REP contained a ticket with the service principal:

```text
ldap/dc01.corp.test
```

The ticket encryption type was AES256-CTS-HMAC-SHA1-96 (etype 18).

The ticket is protected using the long-term key of the account associated with the LDAP SPN.

The outer encrypted portion of the TGS-REP is protected using the TGT session key.

Wireshark displayed a message similar to:

```text
missing keytype 18 usage 8 or 9
```

This indicates that Wireshark did not have the required key to decrypt that portion of the response. It does not, by itself, indicate that Kerberos authentication failed.

<img width="1113" height="991" alt="image" src="https://github.com/user-attachments/assets/38f61758-64d6-493d-b32f-29c9e07303f7" />


## 6. LDAP Communication and Authentication

### 6.1 Performing an LDAP Directory Search

**Execution location:** CLIENT01-NEW – PowerShell

```powershell
([adsisearcher]"(objectClass=domainDNS)").FindOne().Properties.distinguishedname
```

**Purpose:**

This command searches Active Directory for an object matching the LDAP filter:

```text
(objectClass=domainDNS)
```

It then retrieves the object's Distinguished Name (DN).

**Observed result:**

```text
DC=corp,DC=test
```

This confirms that the LDAP directory query returned the domain's Distinguished Name.

<img width="1233" height="94" alt="image" src="https://github.com/user-attachments/assets/6fe6e125-04c4-4285-8e0e-125edcdbb9b1" />


### 6.2 Inspecting the LDAP Bind Request

Wireshark was used to inspect the LDAP BindRequest.

The authentication mechanism was displayed as:

```text
GSS-SPNEGO
```

SPNEGO (Simple and Protected GSS-API Negotiation Mechanism) allows the client and server to negotiate an authentication mechanism.

The advertised mechanism list included Kerberos-related OIDs and other mechanisms, including NTLMSSP.

The presence of GSS-SPNEGO alone does not prove that Kerberos was selected. The actual authentication token must also be inspected.

<img width="1129" height="843" alt="image" src="https://github.com/user-attachments/assets/5e262cde-e476-4975-ad4a-5a200f903f0a" />



### 6.3 Inspecting the LDAP Bind Response

The LDAP BindResponse contained:

```text
resultCode: success (0)
```

This confirms that the LDAP Bind operation succeeded.

The response also contained `serverSaslCreds`, which was inspected using Wireshark's Packet Bytes pane.

The hexadecimal data was copied and analysed with assistance from ChatGPT.

<img width="1138" height="627" alt="image" src="https://github.com/user-attachments/assets/52a47a40-54e8-4e90-b55e-cce4e4544b9b" />


## 7. AP Exchange – Inspecting AP-REQ and AP-REP

### 7.1 Identifying the AP-REP

The server's `serverSaslCreds` field contained a SPNEGO response token.

The following byte sequence was identified:

```text
A1 81 B6
```

This marks the beginning of the SPNEGO NegTokenResp structure in the captured response.

The negotiation state contained:

```text
A0 03 0A 01 00
```

This represents:

```text
negState: accept-completed
```

The response also contained the following Kerberos V5 GSS-API mechanism OID:

```text
1.2.840.113554.1.2.2
```

Within the response token, the following identifier was identified:

```text
02 00
```

This is the Kerberos GSS-API AP-REP token identifier.

The AP-REP is the server's response used in Kerberos mutual authentication.

**Screenshot – AP-REP in LDAP BindResponse**

The following screenshot shows the LDAP BindResponse, the `serverSaslCreds` field, and the corresponding hexadecimal data inspected in Wireshark.

<img width="1270" height="759" alt="image" src="https://github.com/user-attachments/assets/3c96c086-4d49-4a07-a627-83883bec86dd" />


### 7.2 Identifying the AP-REQ

The client's LDAP BindRequest contained SASL credentials with a GSS-SPNEGO token.

The following structure was identified in the captured hexadecimal data:

```text
60 82 07 68
06 09 2A 86 48 86 F7 12 01 02 02
01 00
6E 82 07 57
```

The token contained:

- A GSS-API InitialContextToken.
- A Kerberos V5 GSS-API mechanism OID.
- The Kerberos AP-REQ token identifier `01 00`.
- The ASN.1 APPLICATION 14 tag `6E`, identifying the Kerberos AP-REQ message structure.

This provided direct evidence that the client's LDAP BindRequest contained a Kerberos AP-REQ token.

**Screenshot – AP-REQ in LDAP BindRequest**

The following screenshot shows the LDAP BindRequest, the SASL credentials, and the hexadecimal data containing the AP-REQ token.

<img width="1267" height="1143" alt="image" src="https://github.com/user-attachments/assets/80711e19-c18e-4579-abc5-6aeda0322332" />


### 7.3 AP-REQ and AP-REP Comparison

| Token | Identifier | Direction | Purpose |
|---|---|---|---|
| AP-REQ | `01 00` | Client → Server | Presents the Service Ticket and authenticator. |
| AP-REP | `02 00` | Server → Client | Supports mutual authentication. |

The AP-REQ and AP-REP were identified in the LDAP SASL exchange.

The encrypted ticket and authenticator contents were not decrypted during this investigation.

## 8. Use of AI Assistance

ChatGPT was used as an analytical aid during the investigation.

The use of AI was particularly relevant when interpreting binary data that could not be fully understood through the Wireshark interface alone.

The assistance included:

- Explaining the structure of SPNEGO and GSS-API tokens.
- Interpreting ASN.1 tag and length fields.
- Decoding and identifying OIDs.
- Identifying the AP-REQ and AP-REP token identifiers.
- Helping determine whether the captured LDAP exchange provided evidence of a completed AP authentication exchange.
- Explaining the distinction between the AP-REQ inside PA-TGS-REQ and the AP-REQ presented to the LDAP service.

**Important limitation:**

The binary analysis was not performed independently by the author. ChatGPT assisted with interpreting the hexadecimal data and explaining the protocol structures.

The author performed the lab configuration, generated the network traffic, captured the packets, selected the relevant fields, and reviewed the resulting evidence.

The interpretations were used as guidance for understanding the captured protocol data. Further independent practice with ASN.1, BER, and protocol specifications is required to develop the ability to perform this analysis without AI assistance.

## 9. Results and Conclusion

The investigation successfully captured and identified the major stages of Kerberos authentication in a Windows Server 2025 Active Directory environment.

The following results were obtained:

1. The AS Exchange was observed, including the pre-authentication challenge and TGT issuance.
2. The TGS Exchange was observed, and an LDAP Service Ticket was issued.
3. An LDAP directory search was successfully performed.
4. The LDAP SASL exchange was inspected using Wireshark.
5. The client's AP-REQ and the server's AP-REP were identified in the captured LDAP exchange.
6. The LDAP BindResponse returned `success (0)`.

This lab provided practical experience in:

- Windows Active Directory authentication.
- Kerberos ticket acquisition and service authentication.
- LDAP directory queries.
- Wireshark packet capture and protocol analysis.
- SPNEGO and GSS-API token inspection.
- Interpreting
