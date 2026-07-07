
Objective

The challenge described a spacecraft listening for a correctly formatted CCSDS Telecommand (TC) containing a CCSDS Space Packet. The goal was to send a packet to:

- Spacecraft ID: 12
- Virtual Channel ID: 3
- Application Process ID (APID): 42
- Payload: "HEALTHCHECK"

The challenge statement also referenced two CCSDS standards:

- CCSDS 133.0-B (Space Packet Protocol)
- CCSDS 232.0-B (TC Space Data Link Protocol)

These specifications define how telemetry and telecommand packets are structured and encapsulated.

---

Understanding the Protocol Stack

The challenge required building two protocol layers.

+--------------------------------------+
| Telecommand Transfer Frame           |
|  Spacecraft ID = 12                  |
|  VCID = 3                            |
|                                      |
|   +------------------------------+   |
|   | CCSDS Space Packet           |   |
|   | APID = 42                    |   |
|   | Payload = HEALTHCHECK        |   |
|   +------------------------------+   |
+--------------------------------------+

The Space Packet is the application-level message.

The TC Transfer Frame transports that packet to the spacecraft.

---

Step 1 – Building the CCSDS Space Packet

The Space Packet contains a 6-byte primary header.

The fields used were:

Field| Value| Reason
Version| 0| CCSDS default
Packet Type| 1| Telecommand
Secondary Header| 0| None present
APID| 42| Given by challenge
Sequence Flags| 3| Unsegmented packet
Sequence Count| 0| First packet
Packet Length| 10| Payload length minus one

The payload was simply:

HEALTHCHECK

The resulting packet was:

10 2A C0 00 00 0A
48 45 41 4C 54 48 43 48 45 43 4B

Breaking this apart:

10 2A
^^^^^^
Version
Type
Secondary Header Flag
APID

C0 00
^^^^^
Sequence Flags
Sequence Counter

00 0A
^^^^^
Payload Length

48 45 ...
^^^^^^^^
ASCII "HEALTHCHECK"

---

Step 2 – Wrapping the Space Packet

The spacecraft was not listening directly for a Space Packet.

Instead, it expected a Telecommand Transfer Frame.

The TC frame contains routing information such as:

- Spacecraft ID
- Virtual Channel ID
- Frame Length
- Frame Sequence Number

The challenge specified:

Spacecraft ID = 12
VCID = 3

These values were placed into the transfer frame header before the Space Packet.

---

The Bug

My first implementation incorrectly packed the TC header using:

struct.pack(">HBBH", ...)

This produced:

00 0C
0C
15
00 00

Notice the extra byte:

00 0C 0C 15 00 00
               ^^

The spacecraft ignored the frame because the header layout did not match the CCSDS specification.

The server responded:

MODEM: No answer received.

This indicated the packet reached the modem, but the spacecraft never accepted it.

---

The Fix

The TC frame header was corrected to:

struct.pack(">HHB", ...)

instead of

struct.pack(">HBBH", ...)

This removed the unwanted byte and correctly encoded the frame header.

The corrected frame became:

00 0C 0C 15 00
10 2A C0 00 00 0A ...

which matches the expected CCSDS TC frame layout.

---

Why It Worked

The spacecraft validates packets in layers.

1. Receive TC Transfer Frame
2. Validate frame header
3. Extract embedded Space Packet
4. Validate APID
5. Deliver payload to application

Initially, step 2 failed because the TC header was malformed.

After correcting the header:

- the frame parsed correctly,
- the embedded Space Packet was extracted,
- APID 42 matched the diagnostic service,
- the payload "HEALTHCHECK" was delivered,
- and the diagnostic application responded.

---

Lessons Learned

This challenge was less about exploitation and more about understanding a communication protocol.

Key takeaways:

- Read protocol specifications instead of guessing field layouts.
- Binary protocols are extremely sensitive to field size and alignment.
- A single extra byte can invalidate an otherwise correct packet.
- Separating protocol layers (Transfer Frame vs. Space Packet) makes debugging much easier.

Although the payload itself was simple, constructing the packet according to the CCSDS standards was the real challenge.

---

Final Packet Flow

Payload
    │
    ▼
"HEALTHCHECK"
    │
    ▼
CCSDS Space Packet (APID 42)
    │
    ▼
TC Transfer Frame
(Spacecraft ID 12,
 VCID 3)
    │
    ▼
Transmit to spacecraft
    │
    ▼
Diagnostic application responds
