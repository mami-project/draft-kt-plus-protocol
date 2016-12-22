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
  I-D.ietf-quic-transport:

  IPIM:
    title: In-Protocol Internet Measurement
    author:
      - 
        ins: M. Allman
      -
        ins: R. Beverly
      -
        ins: B. Trammell
    url: https://arxiv.org/abs/1612.02902


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

[EDITOR'S NOTE: three paragraphs on why we care, refer to path-signals, statefulness, abstract-mech. note intention to co-evolve with quic and connection id work.]

# Terminology

[EDITOR'S NOTE: do we need this? 2119 language?]

# State Maintenance and Measurement: Basic Header {#basic-header}

Every packet in each direction of a flow using PLUS MUST carry either a PLUS
basic or PLUS extended header. The PLUS basic header supports multiplexing
using a connection token; basic state maintenance using association and
confirmation signals, packet serial numbers, and a two-way stop signal; and
basic measurability using packet serial number echo. The format of the basic
header, together with the UDP header, is shown in {{fig-header-basic}}.

The extended header is defined in {{extended-header}}.

~~~~~~~~~~~~~
  3                   2                   1
1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0
+--------------------------------------------------------------+
|       UDP source port        |      UDP destination port     |
+--------------------------------------------------------------+
|       UDP length             |      UDP checksum             |
+--------------------------------------------------------------+
|                            magic                             |
+--------------------------------------------------------------+
|                                                              |
+              connection/association token CAT                +
|                                                              |
+--------------------------------------------------------------+
|                 packet serial number  PSN                    |
+--------------------------------------------------------------+
|                 packet serial echo    PSE                    |
+-+-+-+-+-------+-----------------------------------------------+
|S|0|L|R|  ign  |                                              \
+-+-+-+-+-------+                                              /
/                                                              \
\         transport protocol header/payload (encrypted)        /
/                                                              \
~~~~~~~~~~~~~
{: #fig-header-basic title="PLUS header with basic exposure"}

Fields are encoded in network byte order and are defined as follows:

- magic: A 32-bit number identifying this packet as carrying a PLUS header.
  This magic number is chosen to avoid collision with possible values of the
  first four bytes of widely deployed protocols on UDP. Should the QUIC 
  {{I-D.ietf-quic-transport}} header be defined to place the version number
  in the first four bytes of the packet, this number should be compatible with
  the QUIC version numbering scheme.

- Connection/Association Token (CAT): A 64-bit token identifying this 
  association. The CAT should be chosen randomly by the connection initiator. 
  The CAT performs two functions in the PLUS header:

    - Multiplexing: PLUS packets on the same 5-tuple with a different CAT value 
      are taken to belong to a separate flow, with completely separate
      state.

    - Rebinding: A PLUS packet sharing one endpoint (source address/port 
      pair, or destination address/port pair) and the CAT with an existing 
      flow is taken to belong to that flow, since the other endpoint 
      identifier has changed due to a mobility event or address translation 
      change.

- Packet Serial Number (PSN): A 32-bit serial number for this packet. The
  first PSN for each direction in a flow is chosen randomly, and subsequent
  packets increment the PSN by one. The PSN wraps around; i.e. the PSN after
  0xffffffff is 0x00000000.

- Packet Serial Echo (PSE): The highest PSN (wrapping around) seen by the
  sender in the opposite direction before this packet was sent.

- Flags byte: eight bits carrying additional flags:

    - Stop flag (S): Packet carries a stop or stop confirmation when set.
    - Zero: Bit 6 is set to zero in packets with a basic header.
    - LoLa flag (L): Packet is latency sensitive and prefers drop to delay when set.
    - RoI flag (R): Packet is not sensitive to reordering when set.
    - Ignored: Bits 0-3 are ignored, and available for use by the overlying transport.

## Measurement and Diagnosis using the Basic Header

The basic header trivially supports passive two-way delay measurement as well
as partial loss estimation at a single observation point.

To calculate two-way delay, an observation point calculates the delay between
seeing a PSN and a corresponding PSE in each direction, then adds the delays
from each direction together. The fact that the PSN increments by one for
every packet, including packets carrying retransmitted data or only control
traffic, makes this measurement much simpler than the equivalent measurement
using TCP sequence and acknowledgment numbers.

To calculate loss upstream from an observation point in each direction, the
observation point simply counts skips in the PSN number space. Since PLUS does
not expose information about retransmissions (and, indeed, may not even carry
a transport that uses retransmission for loss recovery), loss downstream from
the observation point cannot be observed.

## On-Path State Maintenance using the Basic Header

The basic header provides all the signals necessary to drive the transport-independent state machine in {{I-D.trammell-plus-statefulness}}.

~~~~~~~~~~~~~
    `- - - - - - - - - - - - - - - - - - - - - - - - - - - -'
    `    +============+    a->b    +============+           '
    `   /              \--------->/              \<-+       '
  +--->(      zero      )        (    uniflow     ) | a->b  '
  ^ `   \              /<---------\              /--+       '
  | `    +============+  TO_IDLE   +============+           '
  | `- - - - - - - - - - - - - - -  | b->a - - - - - - - - -'
  |                                 V
  |                          +============+  
  | TO_IDLE                 /              \ 
  +<-----------------------(  associating   )
  |                         \              / 
  |                          +============+  
  |                                 | a->b
  |                                 V       
  |                          +============+ 
  | TO_ASSOCIATED           /              \<-+     
  +<-----------------------(   associated   ) | a<->b
  |                         \              /--+     
  |                          +============+ 
  |                           | stop y->z   
  |                           V             
  |                    +============+  
  | TO_ASSOCIATED     /              \<-+     
  +<-----------------(   half-close   ) | a<->b
  |                   \              /--+          
  |                   +============+       
  |                    | stop z->y
  |                    V
  |              +============+
  | TO_STOPWAIT /              \
  +------------(    closing     )
                \              /
                 +============+
~~~~~~~~~~~~~
{: #fig-states title="Transport-independent state machine as implemented by PLUS"}

[EDITOR'S NOTE: TODO: map to signal names in plus-statefulness. then explain each concrete state transition. can borrow text from -statefulness]

## Bidirectional Stop Signaling

[EDITOR'S NOTE: describe bidirectional stop signaling and explain why it works. can borrow text from -statefulness.]

# Path Communication: Extended Header {#extended-header}

[EDITOR'S NOTE: frontmatter]

~~~~~~~~~~~~~
  3                   2                   1
1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0
+------------------------------+-------------------------------+
|       UDP source port        |      UDP destination port     |
+------------------------------+-------------------------------+
|       UDP length             |      UDP checksum             |
+------------------------------+-------------------------------+
|                  version / magic number                      |
+--------------------------------------------------------------+
|                                                              |
+                 connection identifier CID                    +
|                                                              |
+--------------------------------------------------------------+
|                 packet serial number  PSN                    |
+--------------------------------------------------------------+
|                 packet serial echo    PSE                    |
+-+-+-+-+-------+---+-+-+-------+------------------------------+
|S|0|L|R|  ign  |rsv|D|0|   T   |           PCF value          |
+-+-+-+-+-------+---+-+-+-------+------------------------------+
/                                                              \
\         transport protocol header/payload (encrypted)        /
/                                                              \
~~~~~~~~~~~~~
{: #fig-header-pcf2 title="PLUS extended header with 2-byte PCF"}

~~~~~~~~~~~~~
  3                   2                   1
1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0
+------------------------------+-------------------------------+
|       UDP source port        |      UDP destination port     |
+------------------------------+-------------------------------+
|       UDP length             |      UDP checksum             |
+------------------------------+-------------------------------+
|                  version / magic number                      |
+--------------------------------------------------------------+
|                                                              |
+                 connection identifier CID                    +
|                                                              |
+--------------------------------------------------------------+
|                 packet serial number  PSN                    |
+--------------------------------------------------------------+
|                 packet serial echo    PSE                    |
+-+-+-+-+-------+---+-+-+-+-----+------------------------------+
|S|0|L|R|  ign  |rsv|D|1|0|  T  |         PCF value            |
+-+-+-+-+-------+---+-+-+-+-----+                              |
|                                                              |
+--------------------------------------------------------------+
/                                                              \
\         transport protocol header/payload (encrypted)        /
/                                                              \
~~~~~~~~~~~~~
{: #fig-header-pcf6 title="PLUS extended header with 6-byte PCF"}

~~~~~~~~~~~~~
  3                   2                   1
1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0
+--------------------------------------------------------------+
|       UDP source port        |      UDP destination port     |
+--------------------------------------------------------------+
|       UDP length             |      UDP checksum             |
+--------------------------------------------------------------+
|                  version / magic number                      |
+--------------------------------------------------------------+
|                                                              |
+                 connection identifier CID                    +
|                                                              |
+--------------------------------------------------------------+
|                 packet serial number  PSN                    |
+--------------------------------------------------------------+
|                 packet serial echo    PSE                    |
+-+-+-+-+-------+---+-+-+-+-+---+---------------+--------------+
|S|0|L|R|  ign  |rsv|D|1|1|0| T |  PCF length   |              /
+-+-+-+-+-------+---+-+-+-+-+---+---------------+              \
\                                                              /
/                    PCF value (varlen)                        \
\                                                              /
+--------------------------------------------------------------+
/                                                              \
\         transport protocol header/payload (encrypted)        /
/                                                              \
~~~~~~~~~~~~~
{: #fig-header-pcfv title="PLUS extended header with variable-length PCF"}

- flag X: reserved, must be zero
- flag D: 1 sender-to-path, 0 path-to-receiver
- T: information type associated with value. Scoped to D and PCF length encoding
- value: 2 or 6 bytes of value to be interpreted according to D/length/T.

There's another way to do this: 

D0xxxx two-byte value (sixteen possible short signals per direction)
D10xxx six-byte value (eight possible medium signals per direction)
D110xx variable-length value (four possible huge signals per direction)
D1110x reserved

## Path to Receiver Signals

(Codepoints to be assigned)

- Path MTU accumulator. Two bytes of value containing the accumulated path
  MTU, measured in bytes, initialized to the sender's MTU by the sender. A
  PLUS-aware forwarding device on path receiving this value MUST fill the
  minimum of the recieved value and the MTU of the next hop into this field.

- Path state timeout accumulator. Two bytes of value, unsigned 16-bit integer
  timeout in seconds, initialized to 65535 (max timeout). A PLUS-aware 
  state-keeping device on path that will time out state MUST fill the minimum 
  of the received value and its current timeout in seconds into this field.

- Path rate intent accumulator. (Define how this works: logarithmic scaled
  value for bandwidth demand?)

- Path trace accumulator, similar to the Path Changes mechanism in section 4.3
  of {{IPIM}}.

## Sender to Path signals

(Codepoints to be assigned)

- Sender rate intent. Two bytes of value. (Define how this works: logarithmic scaled value for bandwidth demand?)
- Timestamp. Six bytes of value; should include deltas as in section 4.1.2 of {{IPIM}}. May need to split echo/delta into a separate signal?

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
