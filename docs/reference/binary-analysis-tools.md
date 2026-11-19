# Binary Analysis Tools and Fundamentals

## 1. Purpose

This document provides a simple introduction to binary data analysis and the tools that can assist with it.

The goal is **not to analyse every byte manually**. The goal is to understand basic data structures, use appropriate tools, and interpret the results correctly.

These skills are useful when investigating protocols such as Kerberos, LDAP, DNS, TLS, and other network communications.

## 2. Basic Concepts

| Concept | Description |
|---|---|
| Binary | Data represented using 0 and 1 |
| Bit | A single binary digit (0 or 1) |
| Byte | A group of 8 bits |
| Hexadecimal | Base-16 representation using 0–9 and A–F |
| ASCII | A character encoding that represents text using numeric values |
| Hex Stream | A continuous sequence of hexadecimal values |
| ASN.1 | A notation for defining structured data |
| BER | Rules for encoding ASN.1 data |
| TLV | Tag, Length, Value: a common way to structure encoded data |
| OID | An identifier used to identify standards, protocols, and mechanisms |
| SPNEGO | A mechanism for negotiating an authentication method |
| GSS-API | An interface for security services, including authentication |
| Kerberos Token | A token such as AP-REQ or AP-REP used in Kerberos authentication |

## 3. Recommended Analysis Tools

### 3.1 Wireshark

**Purpose:** Capture and inspect network packets.

**When to use:**
- Identify network protocols.
- Inspect packet fields and protocol structures.
- Follow authentication exchanges.
- Examine raw packet bytes.

**Useful features:**
- Packet Details: Shows decoded protocol fields.
- Packet Bytes: Displays packet data in hexadecimal and ASCII.
- Display Filters: Helps isolate specific protocols.

Official website: https://www.wireshark.org/

### 3.2 CyberChef

**Purpose:** Convert, decode, and manipulate data.

**When to use:**
- Convert hexadecimal values into bytes or text.
- Decode Base64.
- Convert between different data representations.
- Inspect and transform data during analysis.

**Example workflow:**
1. Copy a Hex Stream from Wireshark.
2. Paste it into CyberChef.
3. Select an appropriate operation.
4. Inspect the output.
5. Compare the result with the original packet.

Official tool: https://gchq.github.io/CyberChef/

### 3.3 ASN.1 Decoder

**Purpose:** Help interpret ASN.1 structures encoded using BER or DER.

**When to use:**
- Inspect nested ASN.1 structures.
- Identify tags and lengths.
- Understand how encoded data is organised.

Example tool: https://asn1.qkation.com/

**Important:** A decoder may display the structure, but understanding the protocol still requires knowledge of its ASN.1 definitions.

### 3.4 010 Editor (Optional)

**Purpose:** Inspect and edit binary files using a hexadecimal editor.

**When to use:**
- Examine binary files directly.
- Inspect byte offsets.
- Compare binary structures.

Official website: https://www.sweetscape.com/010editor/

This is optional. Wireshark and CyberChef are sufficient for many basic network-analysis tasks.

## 4. Basic Analysis Workflow

Use the following workflow when investigating unfamiliar binary data.

1. **Capture:** Use Wireshark to capture the relevant network communication.
2. **Identify:** Select the packet or protocol field you want to investigate.
3. **Extract:** Copy the relevant bytes or Hex Stream.
4. **Inspect:** Use Wireshark or a hexadecimal viewer to inspect the data.
5. **Decode:** Use CyberChef or an ASN.1 decoder when appropriate.
6. **Interpret:** Identify the structure, fields, OIDs, and protocol-specific values.
7. **Verify:** Compare the interpretation with protocol documentation and the surrounding packet exchange.

Do not assume that every hexadecimal sequence represents readable text. Some values represent lengths, identifiers, encrypted data, or other binary structures.

## 5. Practical Example: Kerberos AP-REQ and AP-REP

During the Windows Server Active Directory lab, Kerberos authentication was investigated using Wireshark.

The analysis included:

- Identifying the SPNEGO mechanism.
- Inspecting ASN.1 structures.
- Identifying Kerberos-related OIDs.
- Finding the GSS-API token identifier.
- Identifying the Kerberos AP-REQ and AP-REP tokens.

Important token identifiers:

| Token | Identifier | Meaning |
|---|---|---|
| AP-REQ | `01 00` | Kerberos application request |
| AP-REP | `02 00` | Kerberos application response |

These identifiers were found inside the GSS-API token structure.

**Note:** Identifying an AP-REQ or AP-REP does not mean that its encrypted contents have been decrypted. The token structure and authentication result must be examined separately.

## 6. Important Limitations

Analysis tools assist with understanding data, but they do not automatically explain everything.

- A hexadecimal decoder does not necessarily identify the protocol.
- An ASN.1 decoder needs the correct encoding rules and may need the relevant schema.
- Encrypted data cannot normally be interpreted without the required cryptographic keys.
- A recognised OID identifies a mechanism or object, but does not by itself prove authentication succeeded.
- Tool output should always be checked against packet context and protocol documentation.

Do not upload confidential packet captures, credentials, authentication tokens, or sensitive organisational data to online analysis services. Prefer local tools for sensitive data.

## 7. Recommended Learning Order

1. Binary, bits, and bytes
2. Hexadecimal representation
3. ASCII and character encoding
4. Hex Stream analysis
5. ASN.1 structure
6. BER and TLV
7. Object Identifiers (OID)
8. SPNEGO
9. GSS-API
10. Kerberos AP-REQ and AP-REP

The Kerberos packet captures from the Active Directory lab can be used as practical examples throughout this learning process.

## 8. References

- Wireshark User Guide: https://www.wireshark.org/docs/wsug_html/
- CyberChef: https://gchq.github.io/CyberChef/
- ASN.1 Decoder: https://asn1.qkation.com/
- ASN.1 Playground: https://asn1.io/asn1playground/

---

**Learning objective:** Understand the basic structure of binary data and become comfortable using analysis tools to investigate unfamiliar network protocols.
