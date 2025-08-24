---
tags:
  - dns
  - packet-structure
  - binary-format
---
## Binary Message Format

All DNS messages (queries and responses) follow identical structure defined in RFC 1035.

### Header Structure (Fixed 12 bytes)

```
 0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                     Query ID                  |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|QR|   OPCODE  |AA|TC|RD|RA|   Z    |   RCODE   |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    QDCOUNT                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    ANCOUNT                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    NSCOUNT                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    ARCOUNT                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

### Header Field Details

| Field | Bits | Purpose |
|-------|------|---------|
| **Query ID** | 16 | Random number to match queries with responses |
| **QR** | 1 | 0=Query, 1=Response |
| **OPCODE** | 4 | 0=Standard query, 1=Inverse query, 2=Status |
| **AA** | 1 | Authoritative Answer flag |
| **TC** | 1 | Truncation flag (message too large for transport) |
| **RD** | 1 | Recursion Desired |
| **RA** | 1 | Recursion Available |
| **Z** | 3 | Reserved (must be zero) |
| **RCODE** | 4 | Response code (0=No error, 3=Name error) |
| **QDCOUNT** | 16 | Number of questions |
| **ANCOUNT** | 16 | Number of answer records |
| **NSCOUNT** | 16 | Number of authority records |
| **ARCOUNT** | 16 | Number of additional records |

## Question Section Format

Domain name encoding uses length-prefixed labels:

### Example: "www.example.com"
```
Encoded: 0x03 "www" 0x07 "example" 0x03 "com" 0x00
         ^     ^      ^       ^       ^     ^      ^
         |     |      |       |       |     |      |
         3    "www"   7   "example"   3   "com"   end
```

### Complete Question Structure
```
QNAME:  Variable length domain name
QTYPE:  16-bit record type (1=A, 28=AAAA, 15=MX)
QCLASS: 16-bit class (1=Internet)
```

## Resource Record Format

Used in Answer, Authority, and Additional sections:

```
NAME:     Variable (with compression pointers)
TYPE:     16-bit record type
CLASS:    16-bit class
TTL:      32-bit time-to-live (seconds)
RDLENGTH: 16-bit data length
RDATA:    Variable record-specific data
```

### Compression Mechanism
Reduces message size by using pointers to previous occurrences:
- **Pointer format**: 0xC0XX (first 2 bits = 11, remaining 14 bits = offset)
- **Example**: 0xC00C points to byte 12 in message

## Actual Packet Examples

### DNS Query for example.com A record
```hex
1a2b 0100 0001 0000 0000 0000 0765 7861
6d70 6c65 0363 6f6d 0000 0100 01
```

**Breakdown**:
- `1a2b`: Query ID (6699 decimal)
- `0100`: Flags (standard query, recursion desired)
- `0001 0000 0000 0000`: 1 question, 0 answers, 0 authority, 0 additional
- `07 "example" 03 "com" 00`: Domain name
- `0001`: A record type
- `0001`: Internet class

### Corresponding DNS Response
```hex
1a2b 8180 0001 0001 0000 0000 0765 7861
6d70 6c65 0363 6f6d 0000 0100 01c0 0c00
0100 0100 0151 8000 0493 b836 74
```

**Response breakdown**:
- `1a2b`: Same query ID
- `8180`: Response flags (QR=1, RA=1)
- `0001 0001 0000 0000`: 1 question, 1 answer, 0 authority, 0 additional
- Question section: Same as query
- `c00c`: Compression pointer to offset 12 (domain name)
- `0001 0001`: A record, Internet class
- `00015180`: TTL = 86400 seconds (1 day)
- `0004`: Data length = 4 bytes
- `93b83674`: IP address 147.184.54.116

## Message Size Considerations

### UDP Limitations
- **Traditional limit**: 512 bytes
- **EDNS0 extension**: Up to 4096 bytes commonly
- **Truncation handling**: TC bit set, client retries over TCP

### TCP Format
- **Difference**: 16-bit length field prepended to each message
- **Usage**: Zone transfers, large responses, UDP fallback

---
**Related**: [[DNS Transport Protocols]] | [[DNS Record Types]] | [[DNSSEC Message Extensions]]