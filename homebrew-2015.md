# Homebrew 2015

| Information   | ---
| ---           | ---                                                    
| Origin        | DMRplus IPSC Protocol for HB repeater (20150726).pdf
| Copyright     | DL5DI, G4KLX, DG1HT 2015
| Last updated  | 26. 7. 2015
| Used by       | BrandMeister

## Usage description

This protocol is meant to be used to pass data between Client (hotspot, repeater) and Master (server connecting multiple
clients together)

Protocol is state-full and both Client and Master are expected to track current state (FSM - Finite State Machine) of
connection and indicate the internal state by providing appropriate responses to requests

Transport is IP/UDP on configurable port (usually 62031)

## 1. Procedures

### 1.1. Login Procedure

Before Client can send or receive any DMR data within connection to Master, it must **successfully** pass Login
Procedure

1. Client sends Login Request (2.1)
2. Master responds whether Login can proceed Affirmative Result (2.3) or Negative Result (2.2)
    1. **PROCEDURE END** In case Negative Result (2.2) is received, next login shall not be tried immediately
    2. In case Affirmative Result (2.3) is received, Client shall respond with Login Challenge Response (2.8)
        1. **PROCEDURE END** Master rejects the password challengea as incorrect with Negative Result (2.2)
        2. Master accepts the password challenge response using Affirmative Result (2.3)
            1. Client responds with self Configuration (2.9)

Only successfull execution of this procedure is in state **2.ii.b.a** other states are to be considered unsuccessfull
and Login procedure shall be retried after timeout

### 1.2. Closing Procedure

In case either Client or Master being shut down or restarted, existing authenticated connections shall be notified

- Master notifies Clients using Master Closing Announcement (2.6)
- Client notifies Master using Client Closing Announcement (2.7)

Response from the other party is not expected and shall not be waited for

### 1.3 Keep-Alive Procedure

Once the Client is authenticated, it must send Keep-Alive periodically to Master, to keep the connection up and running.

Recommended period for Client Keep-Alive Request (2.4) is 60 seconds, recommended connection timeout for Master is 5
minutes (5 missed Client Keep-Alive Requests).

- Client, once authenticated, sends Keep-Alive Request (2.4) to Master
- Master responds to request:
    - Keep-Alive Response (2.5) if the connection is still alive and can be used according to Master
    - Master Negative Result (2.2) if the connection is considered closed, and Client must repeat Login Procedure (1.1)

## 2. Protocol data units

Following are separate UDP datagram contents, send either by Client or Master, depending on context

### 2.1. Client Login Request (RPTL)

| Required / Optional | Information Element | Length (Bits) | Length (Bytes) | Remark
| --- | --- | --- | --- | ---    
| Required | Command prefix | 32 | 4 | ASCII, 4 characters, 'RPTL'
| Required | Repeater ID | 32 | 4 | Unsigned, Big-Endian, 4-byte integer

### 2.2. Master Negative Result (MSTNAK)

Can be either result of:

- forbidden login (RPTL response)
- password incorrect (RPTK response)
- packet (configuration or DMR Data) not accepted by Master (invalid or unacceptable)

| Required / Optional | Information Element | Length (Bits) | Length (Bytes) | Remark
| --- | --- | --- | --- | ---    
| Required | Command prefix            |   48              |   6               |   ASCII, 6 characters, 'MSTNAK'
| Required | Repeater ID               |   32              |   4               |   Unsigned, Big-Endian, 4-byte integer

### 2.3. Master Affirmative Result (MSTACK)

Can be either result of:

- login request (RPTL response)
- password accepted (RPTK response)

| Required / Optional | Information Element | Length (Bits) | Length (Bytes) | Remark
| --- | --- | --- | --- | --- 
| Required | Command prefix | 48 | 6 | ASCII, 6 characters, 'MSTACK'
| Required | Repeater ID | 32 | 4 | Unsigned, Big-Endian, 4-byte integer
| Optional | Random challenge | 32 | 4 | Unsigned, Big-Endian, 4-byte integer, see **NOTE-1**

**NOTE-1**: Present only on RPTL response, in other cases packet length shall be only 10 bytes

### 2.4. Keep-Alive Request (MSTPING)

Request shall be sent by Client to Master, usually every 60 seconds. If not sent in expected interval, Master shall
consider Client disconnected, and will require Login Procedure to be executed before next transmission

Expectable results are either:

- Keep-Alive Response (RPTPONG) in case the Client is still considered by Master as connected and authenticated
- Master Negative Result (MSTNAK) in case the connection timed-out or Client is no longer accepted by Master for any
  reason

| Required / Optional | Information Element | Length (Bits) | Length (Bytes) | Remark
| --- | --- | --- | --- | --- 
| Required | Command prefix | 56 | 7 | ASCII, 7 characters, 'MSTPING'
| Required | Repeater ID | 32 | 4 | Unsigned, Big-Endian, 4-byte integer

### 2.5. Keep-Alive Response (RPTPONG)

Master shall respond to Keep-Alive Request (MSTPING) with single Keep-Alive Response (RPTPONG) to confirm the connection
is still valid and Client remains connected

| Required / Optional | Information Element | Length (Bits) | Length (Bytes) | Remark
| --- | --- | --- | --- | --- 
| Required | Command prefix | 56 | 7 | ASCII, 7 characters, 'RPTPONG'
| Required | Repeater ID | 32 | 4 | Unsigned, Big-Endian, 4-byte integer

### 2.6. Master Closing Announcement (MSTCL)

When Master is being shut down or restarted, it should notify all currently connected Clients that the connections are
being invalidated and Login Procedure must be executed before next transmission

| Required / Optional | Information Element | Length (Bits) | Length (Bytes) | Remark
| --- | --- | --- | --- | --- 
| Required | Command prefix | 40 | 5 | ASCII, 5 characters, 'MSTCL'
| Required | Repeater ID | 32 | 4 | Unsigned, Big-Endian, 4-byte integer, see **NOTE-1**

**NOTE-1**: ID of Client whose connection is being terminated by this packet

### 2.7. Client Closing Announcement (RPTCL)

When Client is being shut down or restarted, it should notify all Masters it's currently connected to, about connection
being invalidated and that the Client is not available until Login Procedure is executed again

| Required / Optional | Information Element | Length (Bits) | Length (Bytes) | Remark
| --- | --- | --- | --- | --- 
| Required | Command prefix | 40 | 5 | ASCII, 5 characters, 'RPTCL'
| Required | Repeater ID | 32 | 4 | Unsigned, Big-Endian, 4-byte integer

### 2.8 Client Login Challenge Response (RPTK)

As a part of Login Procedure, Client shall use "Random challenge" from (2.3) and prove knowledge of password by
responding with appropriate Challenge Response

| Required / Optional | Information Element | Length (Bits) | Length (Bytes) | Remark
| --- | --- | --- | --- | --- 
| Required | Command prefix | 32 | 4 | ASCII, 4 characters, 'RPTK'
| Required | Repeater ID | 32 | 4 | Unsigned, Big-Endian, 4-byte integer
| Required | Challenge response | 256 | 32 | SHA-256, see **NOTE-1**
| - | - | 320 | 40 | Total size of PDU

**NOTE-1**:

- SHA-256 produces 256-bits of hash, which is sent raw
- Response is calculated from login challenge (2.3 "Random challenge") bytes and password bytes being put side-by-side
    - `sha256( concat( login_challenge_bytes , password_bytes ) )`

### 2.9 Client Configuration (RPTC)

As a part of Login Procedure, Client shall sent its configuration to Master after proving knowledge of correct password

| Required / Optional | Information Element | Length (Bits) | Length (Bytes) | Remark
| --- | --- | --- | --- | --- 
| Required | Command prefix | 32 | 4 | ASCII, 5 characters, 'RPTC'
| Required | Callsign | 64 | 8 | ASCII, 8 characters, see **NOTE-1**
| Required | Repeater ID | 32 | 4 | Unsigned, Big-Endian, 4-byte integer
| Required | RX Frequence | 72 | 9 | ASCII, numbers only, 9 characters, frequence in Hz
| Required | TX Frequence | 72 | 9 | ASCII, numbers only, 9 characters, frequence in Hz
| Required | TX Power | 16 | 2 | ASCII, numbers only, 2 characters, power in dBm, 00-99
| Required | Color Code | 16 | 2 | ASCII, numbers only, 2 characters, see **NOTE-2**
| Required | Latitude | 64 | 8 | ASCII, numbers with sign (+/-) and dot (.) characters, see **NOTE-3**
| Required | Longitude | 72 | 9 | ASCII, numbers with sign (+/-) and dot (.) characters, see **NOTE-3**
| Required | Height | 24 | 3 | ASCII, numbers, antenna height above ground level in meters
| Required | Location | 160 | 20 | ASCII, 20 characters, see **NOTE-1**
| Required | Description | 160 | 20 | ASCII, 20 characters, see **NOTE-1**
| Required | URL | 992 | 124 | ASCII, 124 characters, see **NOTE-1**
| Required | Software ID | 320 | 40 | ASCII, 40 characters, see **NOTE-1**
| Required | Package ID | 320 | 40 | ASCII, 40 characters, see **NOTE-1**
| - | - | 2416 | 302 | Total size of PDU

**NOTE-1**:

- ASCII fields, also called "free form texts", are to be padded from right side, to be the exact length as listed above,
  eg. Callsign element can contain "DL5DI   " (3 spaces at the end)
- These fields must not contain non-printable ASCII characters, advertisement or country specific characters

**NOTE-2**:

- Color Code accepted valid values are 01 - 15

**NOTE-3**

- Location information (Latitude, Longitude) shall be provided in "Decimal Degrees" format without degree symbol
- eg. Latitude: "51.500843" and Longitude: "-0.126443"

### 2.10 DMR Data (DMRD)

Both Client and Master can send DMR Data PDU to the other party.

If Client is un-authenticated or not allowed to submit DMR Data, Master shall respond with Negative Result (2.2),
otherwise, there shall not be confirmation of PDU receival

| Required / Optional | Information Element | Length (Bits) | Length (Bytes) | Remark
| --- | --- | --- | --- | --- 
| Required | Command prefix | 32 | 4 | ASCII, 4 characters, 'DMRD'
| Required | Sequence counter | 8 | 1 | Unsigned integer, counter of sent DMRD packets, overflowing to zero when reaching 256 (0xFF)
| Required | Source Unit ID | 24 | 3 | Unsigned, Big-Endian, 3-byte integer
| Required | Target Unit ID | 24 | 3 | Unsigned, Big-Endian, 3-byte integer
| Required | Repeater ID | 32 | 4 | Unsigned, Big-Endian, 4-byte integer
| Required | Timeslot | 1 | 0 | Timeslot 1 = 0, Timeslot 2 = 1
| Required | Call Type | 1 | 0 | Group Call = 0, Private (Unit-to-Unit) Call = 1
| Required | Frame Type | 2 | 0 | see **NOTE-1**
| Required | Data Type | 4 | 0 | see **NOTE-2**
| Required | Stream ID | 32 | 4 | Unsigned, Big-Endian, 4-byte integer, see **NOTE-3**
| Required | DMR Data | 264 | 33 | Raw, on-air DMR data, see **NOTE-4**

**NOTE-1**:

Valid values for Frame Type field are:

- '00' - Voice Data
- '01' - Voice Sync
- '10' - Data or Data Sync
- '11' - Unused

**NOTE-2**:

- When Frame Type is 'Data or Data Sync', this is Data Type from Slot Type
- When Frame Type is 'Voice Data' or 'Voice Sync', this is the voice sequence number, with '0000' being 'A' from DMR
  specification, '0001' being 'B' and so on

see section 5.1.2.1 (Figure 5.3, page 29) of ETSI TS 102 361-1 V2.5.1 (2017-10)

**NOTE-3**:

Stream ID should be unique number, can be either random or incremented number, that must remain same from PTT-press to
PTT-release

**NOTE-4**:

The on-air DMR data with possible FEC fixes to the AMBE data and/or Slot Type and/or EMB, etc.

## Examples

### 3.1 RPTL PDU (2.1)

Full PDU bytes: `5250544c00002211`

| Bytes | Decoded | Note |
| --- | --- | --- |
| 5250544c | RPTL | ASCII text |
| 00002211 | 8721 | Repeater ID |

### 3.2 MSTNAK PDU (2.2)

Full PDU bytes: `4d53544e414b00002211`

| Bytes | Decoded | Note |
| --- | --- | --- |
| 4d53544e414b | MSTNAK | ASCII text |
| 00002211 | 8721 | Repeater ID |

### 3.3 MSTACK PDU (2.3)

Full PDU bytes: `4d535441434b000401780a7ed498`

| Bytes | Decoded | Note |
| --- | --- | --- |
| 4d535441434b |  | ASCII text |

### 3.4 MSTPING PDU (2.4)

Full PDU bytes: ``

| Bytes | Decoded | Note |
| --- | --- | --- |
|  |  | ASCII text |

### 3.5 RPTPONG PDU (2.5)

Full PDU bytes: ``

| Bytes | Decoded | Note |
| --- | --- | --- |
|  |  | ASCII text |

### 3.6 MSTCL PDU (2.6)

Full PDU bytes: ``

| Bytes | Decoded | Note |
| --- | --- | --- |
|  |  | ASCII text |

### 3.7 RPTCL PDU (2.7)

Full PDU bytes: ``

| Bytes | Decoded | Note |
| --- | --- | --- |
|  |  | ASCII text |

### 3.8 RPTK PDU (2.8)

Full PDU bytes: ``

| Bytes | Decoded | Note |
| --- | --- | --- |
|  |  | ASCII text |

### 3.9 RPTC PDU (2.9)

Full PDU bytes: ``

| Bytes | Decoded | Note |
| --- | --- | --- |
|  |  | ASCII text |

### 3.10 MSTPING PDU (2.10)

Full PDU bytes: ``

| Bytes | Decoded | Note |
| --- | --- | --- |
|  |  | ASCII text |
