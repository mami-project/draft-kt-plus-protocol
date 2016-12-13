---
title: Path Layer UDP Substrate Protocol Specification
abbrev: PLUS Protocol
docname: draft-kt-plus-protocol-00
date:
category: exp

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: B. Trammell
    name: Brian Trammell
    org: ETH Zurich
    email: ietf@trammell.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland
  -
    ins: M. Kuehlewind
    name: Mirja Kuehlewind
    org: ETH Zurich
    email: mirja.kuehlewind@tik.ee.ethz.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland

informative:
  RFC0793:
  RFC2474:
  RFC7675:

  I-D.hardie-path-signals:
  I-D.trammell-plus-abstract-mech:
  I-D.trammell-plus-statefulness:

normative:
  RFC5103:
  RFC7011:
  RFC7398:

--- abstract

This document specifies a Path Layer UDP Substrate (PLUS) protocol for
providing a common wire image for encrypted transport protocols carried over
UDP. The base PLUS header carries information for driving a minimal state
machine at middleboxes described in {{I-D.trammell-plus-statefulness}}, and
provides optional exposure of additional information to devices along the path
using the mechanism described in {{I-D.trammell-plus-abstract-mech}}.

--- middle

# Introduction

[EDITOR'S NOTE: three paragraphs on why we care, refer to path-signals, statefulness, abstract-mech. note intention to co-evolve with plus and connection id work.]

# Terminology

[EDITOR'S NOTE: do we need this? 2119 language?]

# Base Header

~~~~~~~~~~~~~

  3                   2                   1
1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0
+--------------------------------------------------------------+
|       UDP source port        |      UDP destination port     |
+--------------------------------------------------------------+
|       UDP length             |      UDP checksum             |
+--------------------------------------------------------------+
|                 version / magic number A                     |
+-----+--------+-----------------------------------------------+
|  0  |ignored |                                               \
+-----+--------+                                               /
/                                                              \
\         transport protocol header/payload (encrypted)        /
/                                                              \
~~~~~~~~~~~~~
{: #fig-header-blank title="PLUS header with no exposure"}

~~~~~~~~~~~~~
  3                   2                   1
1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0
+--------------------------------------------------------------+
|       UDP source port        |      UDP destination port     |
+--------------------------------------------------------------+
|       UDP length             |      UDP checksum             |
+--------------------------------------------------------------+
|                 version / magic number B                     |
+--------------------------------------------------------------+
|                 connection identifier CID                    |
+--------------------------------------------------------------+
|                 packet serial number  PSN                    |
+--------------------------------------------------------------+
|                 packet serial echo    PSE                    |
+-+-+----------+-----------------------------------------------+
|S|0| ignored  |                                               \
+-+-+----------+                                               /
/                                                              \
\         transport protocol header/payload (encrypted)        /
/                                                              \
~~~~~~~~~~~~~
{: #fig-header-basic title="PLUS header with basic exposure"}


[EDITOR'S NOTE: prosify the following]

The PLUS header appears on each packet. It has the following fields:

- magic: "compatible" with QUIC version number, must have non-reflectable property identified in SPUD.
- CID: refer to Connection ID design team output for DTLS/QUIC
- PSN: initial chosen randomly, increments by one for each packet sent, including control-only packets and retransmissions.
- PSE: highest PSN seen when packet sent, for RTT and state establishment
- PCL: path control length code: 00 = not present; 01 = 3 byte; 10 = 7 byte; 11 = reserved.
- S: bidirectional stop, see {{bidirectional-stop-signaling}}
- ignored flags: reserved for use by overlying transport protocol
- PCF: see definition in {{path-communication-field}} 

## Measurement and Diagnosis using the Basic Header

[EDITOR'S NOTE: explain how to measure RTT and loss using CID/PSN/PSE]

## On-Path State Maintenance using the Basic Header

[EDITOR'S NOTE: note rough TCP-equivalence of this state machine...]

~~~~~~~~~~~~~
redraw this for bidirectional stop and specific CID/PSN/PSE signaling

      .- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -.
      '    +==============+    pkt(s->d)    +==========+              '
      '   //              \\-------------->/            \--+          '
      '  ((      zero      ))             (    uniflow   ) |pkt(s->d) '
      '   \\              //<--------------\            /<-+          '
      '    +==============+  TO_IDLE/close  +==========+              '
      '- - -|- - -  ^ - ^  - - - - - - - - - - - - - -|- - - - - - - -'
            |        \   \                            |  association
 TO_CLOSING |         \   \                           V  signal
      +==========+     \   \      TO_IDLE        +==========+  
     /            \     \   +-------------------/            \--+ 
    (    closing   )     \                     (  associating ) | pkt
     \            /       \                     \            /<-+(s->d)
      +==========+         \ TO_ASSOCIATED       +==========+  
            ^               \                         |
            |               +==========+              |  
     close  |              /            \             |  confirmation
    signal  |             (  associated  )            |  signal
            +--------------\            /<------------+
                            +==========+  
                              |      ^
                              +------+
                             pkt(s<->d)
    
~~~~~~~~~~~~~
{: #fig-states title="Signals PLUS provides to the transport-independent state machine"}

## Bidirectional Stop Signaling

[EDITOR'S NOTE: describe bidirectional stop signaling and explain why it works.]

# Path Communication Field

~~~~~~~~~~~~~
  3                   2                   1
1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0
+--------------------------------------------------------------+
|       UDP source port        |      UDP destination port     |
+--------------------------------------------------------------+
|       UDP length             |      UDP checksum             |
+--------------------------------------------------------------+
|                 version / magic number B                     |
+--------------------------------------------------------------+
|                 connection identifier CID                    |
+--------------------------------------------------------------+
|                 packet serial number  PSN                    |
+--------------------------------------------------------------+
|                 packet serial echo    PSE                    |
+-+-+----------+-+-+-+-+-+-----+-------------------------------+
|S|1| ignored  |L|R|X|D|0|  T  |          value                |
+-+-+----------+-+-+-+-+-+-----+-------------------------------+
/                                                              \
\         transport protocol header/payload (encrypted)        /
/                                                              \
~~~~~~~~~~~~~
{: #fig-header-pcf3 title="PLUS header with 3-byte PCF"}

~~~~~~~~~~~~~
  3                   2                   1
1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0
+--------------------------------------------------------------+
|       UDP source port        |      UDP destination port     |
+--------------------------------------------------------------+
|       UDP length             |      UDP checksum             |
+--------------------------------------------------------------+
|                 version / magic number B                     |
+--------------------------------------------------------------+
|                 connection identifier CID                    |
+--------------------------------------------------------------+
|                 packet serial number  PSN                    |
+--------------------------------------------------------------+
|                 packet serial echo    PSE                    |
+-+-+----------+-+-+-+-+-+-----+-------------------------------+
|S|1| ignored  |L|R|X|D|1|  T  |          value                |
+-+-+----------+-+-+-+-+-+-----+                               |
|                                                              |
+--------------------------------------------------------------+
/                                                              \
\         transport protocol header/payload (encrypted)        /
/                                                              \
~~~~~~~~~~~~~
{: #fig-header-pcf7 title="PLUS header with 7-byte PCF"}

- flag L: if set, packet is latency sensitive and prefers drop to delay
- flag R: if set, packet is not sensitive to reordering, and may be freely reordered [EDITOR'S NOTE: how does this interact with PSN/PSE?]
- flag X: reserved, must be zero
- flag D: 1 sender-to-path, 0 path-to-receiver
- flag V: 0 two-byte value, 1 six-byte value
- T: information type associated with value. Shown as a five-bit value DVT.
- value: 2 or 6 bytes of value, depending on V, to be interpreted according to T.

## Two-byte Path to Receiver signals

- 0x00: reserved
- 0x01: Path MTU accumulator. Two bytes of value containing the accumulated path MTU, measured in bytes, initialized to the sender's MTU by the sender. A PLUS-aware forwarding device on path receiving this value MUST fill the minimum of the recieved value and the MTU of the next hop into this field.
- 0x02: Path state timeout accumulator. Two bytes of value. (Define how this works)
- 0x03: reserved
- 0x04: reserved
- 0x05: reserved
- 0x06: reserved
- 0x07: reserved

# Six-byte Path to Receiver Signals

- 0x08: reserved
- 0x09: reserved
- 0x0a: reserved
- 0x0b: reserved
- 0x0c: reserved
- 0x0d: reserved
- 0x0e: reserved
- 0x0f: reserved

## Two-byte Sender to Path signals

- 0x10: reserved
- 0x11: Sender rate intent. Two bytes of value. (Define how this works: logarithmic scaled value for bandwidth demand.)
- 0x12: reserved
- 0x13: reserved
- 0x14: reserved
- 0x15: reserved
- 0x16: reserved
- 0x17: reserved

## Six-byte Sender to Path signals

- 0x18: reserved
- 0x19: Timestamp. Six bytes of value. (Define how this works)
- 0x1a: reserved
- 0x1b: reserved
- 0x1c: reserved
- 0x1d: reserved
- 0x1e: reserved
- 0x1f: reserved

## Integrity protection

[EDITOR'S NOTE: is it sufficient to delegate this completely to the overlying transport, or do we want to define an HMAC based on a secret coordinated with the overlying transport?]

# IANA Considerations

This document has no actions for IANA. Signal types may be moved to a Standards Action registry in the future.

# Security Considerations

[EDITOR'S NOTE: write me]

# Acknowledgments

This work is partially supported by the European Commission under Horizon 2020
grant agreement no. 688421 Measurement and Architecture for a Middleboxed
Internet (MAMI), and by the Swiss State Secretariat for Education, Research,
and Innovation under contract no. 15.0268. This support does not imply
endorsement.
