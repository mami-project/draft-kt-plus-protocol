---
title: Path Layer UDP Substrate Specification
abbrev: PLUS Spec
docname: draft-trammell-plus-spec-01
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
  RFC4821:
  RFC7675:
  RFC7323:
  RFC8035:

  I-D.hardie-path-signals:
  I-D.trammell-plus-abstract-mech:
  I-D.trammell-plus-statefulness:
  I-D.ietf-quic-transport:

  IPIM:
    title: Principles for Measurability in Protocol Design (arXiv preprint 1612.02902)
    author:
      - 
        ins: M. Allman
      -
        ins: R. Beverly
      -
        ins: B. Trammell
    url: https://arxiv.org/abs/1612.02902
    date: 2016-12-09


--- abstract

This document specifies a common Path Layer UDP Substrate (PLUS) wire image
for encrypted transport protocols carried over UDP. The base PLUS header
carries information for driving a minimal state machine at middleboxes
described in {{I-D.trammell-plus-statefulness}}, and provides optional
exposure of additional information to devices along the path using the
mechanism described in {{I-D.trammell-plus-abstract-mech}}.

--- middle

# Introduction

This document defines a wire image for a Path Layer UDP Substrate (PLUS), for
limited exposure of information from encrypted, UDP-encapsulated transport
protocols. The wire image implements signaling to drive the minimal state
machine defined in {{I-D.trammell-plus-statefulness}} as well as optional
exposure of additional information to devices along the path using the
mechanism described in {{I-D.trammell-plus-abstract-mech}}.

As discussed in {{I-D.hardie-path-signals}}, basic information about flows
currently exposed by TCP are missing from transport protocols that encrypt
their headers. Given the ossification of protocol wire images due to the
widespread deployment of stateful network devices that rely on header
inspection, this header encryption is necessary to support transport protocol
evolution. However, the loss of basic information for on-path state
maintenance as well as network performance measurement, diagnostics, and
troubleshooting via header encryption makes network management more difficult.
The PLUS wire image defined by this document aims to mitigate this difficulty,
allowing deployment of encrypted protocols without loss of essential in-
network functionality.

This wire image is intended primarily to support state maintenance and
measurement; the principles of measurement and primitives we aim to support
are drawn from recent work on explicit measurability in protocol design
{{IPIM}}.

# State Maintenance and Measurement: Basic Header {#basic-header}

Every packet in each direction of a flow using PLUS carries a PLUS header. This
can be either a basic header, or an extended header. The PLUS basic header
supports multiplexing using a connection token; basic state maintenance using
association and confirmation signals, packet serial numbers, and a two-way stop
signal; and basic measurability using packet serial number echo. The format of
the basic header, together with the UDP header, is shown in
{{fig-header-basic}}.

The extended header is defined in {{extended-header}}.

~~~~~~~~~~~~~
  3                   2                   1
1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0
+------------------------------+-------------------------------+
|       UDP source port        |      UDP destination port     |
+------------------------------+-------------------------------+
|       UDP length             |      UDP checksum             |
+------------------------------+-----------------------+-+-+-+-+
|                            magic                     |L|R|S|0|
+------------------------------------------------------+-+-+-+-+
|                                                              |
+-             connection/association token CAT               -+
|                                                              |
+--------------------------------------------------------------+
|                 packet serial number  PSN                    |
+--------------------------------------------------------------+
|                 packet serial echo    PSE                    |
+--------------------------------------------------------------+
/                                                              \
\         transport protocol header/payload (encrypted)        /
/                                                              \
~~~~~~~~~~~~~
{: #fig-header-basic title="PLUS header with basic exposure"}


Fields are encoded in network byte order and are defined as follows:

- magic: A 28-bit number identifying this packet as carrying a PLUS header. This
  magic number is chosen to avoid collision with possible values of the first
  four bytes of widely deployed protocols on UDP. The value 0xd8007ff has been
  provisionally selected for the PLUS magic number based of experience with the
  SPUD prototype, a cursory survey of common UDP protocols, and compatibility
  with {{RFC8035}}.

- flags: four bits carrying additional information:
    - LoLa flag (L): Packet is latency sensitive and prefers drop to delay when set.
    - RoI flag (R): Packet is not sensitive to reordering when set.
    - Stop flag (S): Packet carries a stop or stop confirmation when set.
    - Extended Header bit: Flag bit 0x01 is zero in packets with a Basic Header.

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
  packets increment the PSN by one. The PSN wraps around.

- Packet Serial Echo (PSE): The most recent PSN seen by the
  sender in the opposite direction before this packet was sent.

Since PLUS is designed to be used for UDP-encapsulated, encrypted transport
protocols, overlying transports are presumed to provide encryption and
integrity protection for their own headers. For the sake of efficiency, it is
also assumed that this integrity protection can be extended to the bits in the
PLUS Basic Header.

## Sender Behavior

When a sender has a packet ready to send using PLUS, it determines the values
in the Basic Header as follows:

- The magic number is set to the constant 0xd8007ff.

- If the sender is the flow initiator, and the packet is the first packet in
  the flow, the sender selects a cryptographically random 64-bit number for
  the CAT. When multiplexing, it must ensure the CAT is not already in use for
  the 5-tuple. Otherwise, the sender uses the CAT associated with the flow.

- If the packet is the first packet in the flow in this direction, the sender
  selects a cryptographically random 32-bit number for the PSN. Otherwise, the
  sender adds one to the PSN on the last packet it sent in this flow, and uses
  that value for the PSN. If the last PSN is 0xffffffff, it wraps around,
  setting the PSN to 0x00000001. A PSN of 0x00000000 is never sent.

- If the packet is the first packet in the flow in this direction, the sender
  sets the PSE to 0x00000000. Otherwise it sets the PSE to the PSN of the last
  packet seen in the opposite direction.

- If the overlying transport determines that this packet is loss-insensitive
  but latency-sensitive, the sender sets the L flag.

- If the overlying transport determines that this packet may be freely
  reordered, the sender sets the R flag.

- If the overlying transport determines that the connection is shutting down,
and no further packets will be sent in this direction other than packets part
of this shutdown, the sender sets the S flag; see
{{twostop}} for details.

## Receiver Behavior

When a receiver receives a packet containing a PLUS Basic Header, it processes
the values in the Basic Header as follows:

- It verifies that the magic number is the constant 0xd800fff. If the receiver is expecting a PLUS packet, and it does not see this value, it drops the packet without further processing.

- It verifies the integrity of the information in the PLUS Basic Header, using
  information carried in the overlying transport. Packets failing integrity
  checks SHOULD be dropped, but MAY be further analyzed by the receiver to
  determine the likely cause of verification failure; reaction to the failure is
  transport and implementation specific.

- It stores the PSN to be sent as the PSE on the the next packet it sends in
  the opposite direction.

## On-Path State Maintenance using the Basic Header

The basic header provides all the signals necessary to drive the transport-
independent state machine described in {{I-D.trammell-plus-statefulness}}, as
shown in {{fig-states}}.

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
  |                             | stop y->z   
  |                             V             
  |                    +============+  
  | TO_ASSOCIATED     /              \<-+     
  +<-----------------(    stop-wait   ) | a<->b
  |                   \              /--+          
  |                   +============+       
  |                       | stop z->y
  |                       V
  |              +============+
  | TO_STOPPING /              \
  +------------(    stopping     )
                \              /
                 +============+
~~~~~~~~~~~~~
{: #fig-states title="Transport-independent state machine as implemented by PLUS"}



### State Establishment

On the first packet with a PLUS header forwarded by an on-path device for a
given 5-tuple plus CAT, the device moves that flow from the zero state to the
uniflow state. The device retuens the flow to zero state after not seeing a
packet on the same flow in the same direction with the same CAT within a
timeout interval TO_IDLE. Otherwise, it stays in uniflow state and continues
forwarding packets, as long as it only observes packets in the same direction
as the initial packet. (the a->b direction in {{fig-states}}).

A PLUS-aware on-path device forwarding a packet with a PLUS header with a
reversed 5-tuple and identical CAT (the b->a direction in {{fig-states}}) to a
flow in the uniflow state, moves that flow to the associating state. It then
waits to see a packet with a PSE in the a->b direction equal to the PSN on the
first reverse packet; on receipt of this packet, the device moves the flow to
associated state. Otherwise, it drops state after a timeout interval TO_IDLE.

Once a flow has moved to the associated state, it will remain in that state
for a timeout interval TO_ASSOCIATED. The on-path device forwards any packet
with a PLUS header in either direction for this flow. It resets the
TO_ASSOCIATED timer for every packet it forwards in this state.

### Bidirectional Stop Signaling {#twostop}

A PLUS-aware on-path device forwarding a packet for a flow in the associated
state with an S flag set moves that flow to stop-wait state. It stores the
PSN on the packet causing the transition, and continues forwarding packets as
if in associated state, dropping state on timeout interval TO_ASSOCIATED.

When it sees a packet in the opposite direction with the S flag set and the
PSE set to exactly the stored PSN, it transitions the flow to stopping state.
The device will forward packets in both directions for flows in the stopping
state within a timeout interval TO_STOPPING; these packets will not reset the
timer.

Note that even though the S flag is integrity-protected end to end, a packet
with the S flag set could be forged by one on-path device to drive the flow
into stop-wait state on all downstream devices. However, this forgery is of
severely limited utility. First, it would require coordination between
attackers on both sides of a given on-path device in order to forge a
confirmation of the stop signal -- a flag with the S bit set and a valid PSE
corresponding to the PSN of the first stop signal to drive the flow into
stopping state. Second, the information in the Basic Header on each packet will
drive the state machine into associated state even in the middle of a flow,
enabling fast recovery even in the case of such a coordinated attack.

### State Rebinding

One end of a PLUS association may change its address while maintaining on-path
state; e.g. due to a NAT change. A PLUS-aware on-path device that forwards a
packet for a flow in the zero state, where one of the endpoint identifiers
(address and port) and the CAT, but not the other endpoint identifier, match a
flow in a non-zero state, treats that packet as belonging to the existing flow,
and updates the endpoint identifier.

## Measurement and Diagnosis using the Basic Header

The basic header trivially supports passive two-way delay measurement as well
as partial loss estimation at a single observation point on path.

To calculate two-way delay, an observation point calculates the delay between
seeing a PSN and a corresponding PSE in each direction, then adds the delays
from each direction together. The fact that the PSN increments by one for
every packet makes this measurement much simpler than the equivalent measurement
using TCP sequence and acknowledgment numbers.

[EDITOR'S NOTE: specify this fully.]

Loss and reordering upstream from an observation point in each direction can be
estimated through examination of the PSN sequence observed. A skipped PSN not
seen within a specified interval can be counted as a lost packet, and the
extent of reordering estimated by the degree of skipping seen in those skipped
PSNs that are later observed. Since PLUS does not expose information about
retransmissions (and, indeed, may not even carry a transport that uses
retransmission for loss recovery), loss downstream from the observation point
cannot be observed.

# Path Communication: Extended Header {#extended-header}

Additional facilities for communicating with on-path devices under endpoint
control are provided by the PLUS Extended Header. The extended header shares the
layout of its first 20 bytes with the PLUS Basic Header, except the Extended
Header bit (0x01 on byte 11) is set. As with the Basic Header, overlying
transports are presumed to provide encryption and integrity protection for the
PLUS Extended Header. The Extended Header has in 21-byte, 22-byte, 24-byte, and
variable-length variants, and three to four additional fields:

- C codepoint (PCF class): a two-bit value defining the class of the PCF value,
  and therefore its treatment at middleboxes and endpoints. It has the following values:
    - 00: sender to path signal: the class, length, type, and content of the PCF
      value are integrity protected end-to-end.
    - 01: path to receiver signal: the class, length, and type of the PCF value
      are integrity protected end to end, but not the content of the value; for
      integrity protection purposes, it is assumed to be all zeroes.
    - 10: reserved for a future revision of this specification.
    - 11: reserved for a future revision of this specification.

- L codepoint (PCF length): a two-bit value encoding the length of the PCF
  value. It has the following values:
    - 00: 0 bytes. See {{fig-header-pcf0}} for the header layout. Zero-length
      PCFs are used to carry marking information on a packet or flow in the PCF
      Type field only.
    - 01: 1 byte. See {{fig-header-pcf1}} for the header layout.
    - 10: 3 bytes. See {{fig-header-pcf3}} for the header layout.
    - 11: Variable length; byte 21 contains the length of the PCF value, 0-255.
      See {{fig-header-pcfv}}.

- PCF Type: a 4-bit value defining the type and semantics of the PCF value,
  scoped to the L and C codepoints (i.e., PCF Type 1 with C=00 and L=00 has a
  distinct meaning from PCF Type 1 with C=00 and L=01). The C, L, and Type
  fields are expressed as a single byte in the subsections below. For example,
  0x01 is C=00, L=00, type 1, a zero-length sender-to-path mark; and 0x61 is
  C=01, L=10, type 1, a 3-byte path-to-receiver signal.

- PCF Value: a 1-byte, 3-byte, or variable-length field, according to the value
  of the L codepoint, containing a value of the type described in the PCF Type
  field.

~~~~~~~~~~~~~
  3                   2                   1
1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0
+------------------------------+-------------------------------+
|       UDP source port        |      UDP destination port     |
+------------------------------+-------------------------------+
|       UDP length             |      UDP checksum             |
+------------------------------+-----------------------+-+-+-+-+
|                            magic                     |L|R|S|1|
+------------------------------------------------------+-+-+-+-+
|                                                              |
+-             connection/association token CAT               -+
|                                                              |
+--------------------------------------------------------------+
|                 packet serial number  PSN                    |
+--------------------------------------------------------------+
|                 packet serial echo    PSE                    |
+---+---+-------+---------------+------------------------------+
| C | 0 | Type  |                                              /
+---+---+-------+                                              \
\                                                              /
/         transport protocol header/payload (encrypted)        \
\                                                              /
~~~~~~~~~~~~~
{: #fig-header-pcf0 title="PLUS extended header with 0-byte PCF"}

~~~~~~~~~~~~~
  3                   2                   1
1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0
+------------------------------+-------------------------------+
|       UDP source port        |      UDP destination port     |
+------------------------------+-------------------------------+
|       UDP length             |      UDP checksum             |
+------------------------------+-----------------------+-+-+-+-+
|                            magic                     |L|R|S|1|
+------------------------------------------------------+-+-+-+-+
|                                                              |
+-             connection/association token CAT               -+
|                                                              |
+--------------------------------------------------------------+
|                 packet serial number  PSN                    |
+--------------------------------------------------------------+
|                 packet serial echo    PSE                    |
+---+---+-------+---------------+------------------------------+
| C | 1 | Type  |   PCF value   |                              /
+---+---+-------+---------------+                              \
\                                                              /
/         transport protocol header/payload (encrypted)        \
\                                                              /
~~~~~~~~~~~~~
{: #fig-header-pcf1 title="PLUS extended header with 1-byte PCF"}

~~~~~~~~~~~~~
  3                   2                   1
1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0
+------------------------------+-------------------------------+
|       UDP source port        |      UDP destination port     |
+------------------------------+-------------------------------+
|       UDP length             |      UDP checksum             |
+------------------------------+-----------------------+-+-+-+-+
|                            magic                     |L|R|S|1|
+------------------------------------------------------+-+-+-+-+
|                                                              |
+-             connection/association token CAT               -+
|                                                              |
+--------------------------------------------------------------+
|                 packet serial number  PSN                    |
+--------------------------------------------------------------+
|                 packet serial echo    PSE                    |
+---+---+-------+---------------+------------------------------+
| C | 2 | Type  |   PCF value   |                              /
+---+---+-------+---------------+                              \
\                                                              /
/         transport protocol header/payload (encrypted)        \
\                                                              /
~~~~~~~~~~~~~
{: #fig-header-pcf3 title="PLUS extended header with 3-byte PCF"}

~~~~~~~~~~~~~
  3                   2                   1
1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0
+------------------------------+-------------------------------+
|       UDP source port        |      UDP destination port     |
+------------------------------+-------------------------------+
|       UDP length             |      UDP checksum             |
+------------------------------+-----------------------+-+-+-+-+
|                            magic                     |L|R|S|1|
+------------------------------------------------------+-+-+-+-+
|                                                              |
+-             connection/association token CAT               -+
|                                                              |
+--------------------------------------------------------------+
|                 packet serial number  PSN                    |
+--------------------------------------------------------------+
|                 packet serial echo    PSE                    |
+---+---+-------+---------------+------------------------------+
| C | 3 | Type  |   PCF length  |                              /
+---+---+-------+---------------+      PCF value (varlen)      \
\                                                              /
+--------------------------------------------------------------+
\                                                              /
/         transport protocol header/payload (encrypted)        \
\                                                              /
~~~~~~~~~~~~~
{: #fig-header-pcfv title="PLUS extended header with variable-length PCF"}

Sender to Path signals are generally used to expose information about the
traffic for measurement or diagnostic purposes. Path to Receiver signals
generally take the form of accumulators: initialized to some value by the
sender, and subject to some aggregation function by each on-path device that
understands them. In any case, the information sent and received is to be
treated as advisory only.

Note that since C codepoints 00 and 01 imply identical treatment for zero-length
(L codepoint 00) PCF values, both (i.e., type bytes 0x00 - 0x0f and 0x40 - 0x4f)
are used for sender-to-path packet marking.

## Measurement and Diagnostics using the Extended Header

We have identified the following sender to path signals as potentially useful
for measurement and diagnostic purposes. These signals are advisory only, and
should not be presumed by either the endpoints or devices along the path to
affect forwarding behavior.

- Packet number echo delta time (Type 0x21, 3-byte sender to path) Exposes the
interval between the receipt of the packet whose number appears in the PSE and
the transmission of this packet, as in section 4.1.2 of {{IPIM}}. Together with
analysis of the PSN and PSE sequence, this allows high-precision RTT estimation.
The encoding of this field is TBD.

- Timestamp (Type 0x22, 3-byte sender to path). Similar to TCP timestamps in
{{RFC7323}}, allows constant-rate clock exposure to devices on path. Note that
this is less necessary for RTT measurement of one-sided flows than it is in TCP,
due to the properties of the PSN and PSE values in the Basic Header. [EDITOR'S
NOTE: is this useful enough to keep?]

- Timestamp Echo (Type 0x23, 3-byte sender to path). Echo of the last received
  timestamp, as above. [EDITOR'S NOTE: is this useful enough to keep?]

We have identified the following path to receiver signals as potentially
useful. Note that accumulated values for use at the sender must be fed back to
the sender by the overlying transport, and that the presence of non-PLUS aware
devices on path at breaks in MTU mean that the accumulated value can only be
used as a hint to processes for measurement and discovery of the accumulated
values at the sender.

- State timeout accumulator (Type 0x51, 1-byte path to receiver): This signal
  allows measurement of timeouts from PLUS-aware devices. It is initialized to a
  maximum ("no information") value by the sender. A PLUS-aware forwarding device
  on path receiving this value fills in the minimum of the received value and
  the configured timeout for the flow's present state into this field. The
  encoding of this field is TBD.

- Rate limit accumulator (Type 0x52, 1-byte path to receiver): This signal
  allows exposure of rate limiting along the path. It is initialized to a
  maximum ("no information") value by the sender. A PLUS-aware forwarding device
  on path receiving this value fills in the minimum of the received value and
  the rate limit to which this flow is subject into this field. The encoding of
  this field is TBD.

- MTU accumulator (Type 0x61, 3-byte path to receiver): This signal allows
  measurement of MTU information from PLUS-aware devices. The sender sets the
  initial value to the sender's MTU. A PLUS-aware forwarding device on path
  receiving this value fills in the minimum of the received value and the MTU of
  the next hop, in bytes into this field. The information, when fed back to the sender,
  can be used as a hint for a running PLPMTUD {{RFC4821}} process.

- Trace accumulator (Type 0x71, varlen path to receiver). This signal allows
  exposure of a trace of PLUS-aware devices on path, similar to the Path Changes
  mechanism in section 4.3 of {{IPIM}}. The sender initializes the value to a
  value chosen randomly for the flow; all packets in the flow using path trace
  accumulator must use the same initial value. A PLUS-aware forwarding device on
  path receiving this value fills in the result of XORing the received value
  with a randomly chosen device identifier, which it must use for all path trace
  accumulator signals it participates in. Packets traversing the same set of
  PLUS-aware forwarding devices in the same flow therefore arrive at the
  receiver with the same accumulated value, and changes to the set of devices on
  path can be detected by the receiver.

# IANA Considerations

This document has no actions for IANA. Path communication field types and PLUS
magic numbers may be moved to a Standards Action registry in a future
revision.

# Security Considerations

This document describes the PLUS Basic and Extended Headers, and the protocol
they support. This protocol can be used to expose information to devices along
the path to replace the analysis of transport- and application-layer headers
when those headers are encrypted. Care must be taken in the exposure of such
information to ensure no irrelevant application and/or user confidential
information is exposed.

PLUS itself contains some security-relevant features. In concert with an
encrypted overlying transport, the PLUS Basic and Extended Headers are
integrity-protected to prevent manipulation on-path of any value except Path to
Receiver values; this integrity protection prevents Path to Receiver values
from being injected without explicit sender involvement, or from being stripped
from the PLUS Extended Header.

The CAT and PSE described in {{basic-header}} taken together, provide entropy
to prevent on-path devices from being driven into incorrect states by off-path
attackers. Bidirectional stop signaling as in {{twostop}} requires an on-path
attacker of a given middlebox to forge traffic on both of the middlebox's
interfaces to drive a middlebox to inappropriately drop state for a flow.

# Acknowledgments

This work is partially supported by the European Commission under Horizon 2020
grant agreement no. 688421 Measurement and Architecture for a Middleboxed
Internet (MAMI), and by the Swiss State Secretariat for Education, Research, and
Innovation under contract no. 15.0268. This support does not imply endorsement.
Thanks to Ted Hardie, Joe Hildebrand, Mark Nottingham, and the participants of
the PLUS BoF at IETF 96 in Berlin for input leading to this design; and to Gorry
Fairhurst for the detailed review.
