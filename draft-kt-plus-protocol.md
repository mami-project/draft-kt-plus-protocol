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

# Basic Headers

Every packet in each direction of a flow using PLUS MUST carry either a PLUS basic or PLUS extended header. The PLUS basic header supports multiplexing using a connection token; basic state maintenance using association/confirmation signals, packet serial numbers, and a two-way stop signal; and basic measurability using packet serial number echo. The format of the basic header, together with the UDP header, is shown in {{fig-header-basic}}

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
+              connection/association token CAT                +
|                                                              |
+--------------------------------------------------------------+
|                 packet serial number  PSN                    |
+--------------------------------------------------------------+
|                 packet serial echo    PSE                    |
+-+-+-+-+------+-----------------------------------------------+
|S|0|L|R| ign  |                                               \
+-+-+-+-+------+                                               /
/                                                              \
\         transport protocol header/payload (encrypted)        /
/                                                              \
~~~~~~~~~~~~~
{: #fig-header-basic title="PLUS header with basic exposure"}

The fields are defined as follows:

[EDITOR'S NOTE todo, prosify]

- magic: "compatible" with QUIC version number, must have non-reflectable property identified in SPUD.
- CAT: note we use this both to identify connections (for fast NAT rebinding) as well as an assocation token. Refer to Connection ID design team output for DTLS/QUIC.
- PSN: initial chosen randomly, increments by one for each packet sent, including control-only packets and retransmissions.
- PSE: highest PSN seen when packet sent, for RTT and state establishment
- flag S: bidirectional stop, see {{bidirectional-stop-signaling}}
- flag L: if set, packet is latency sensitive and prefers drop to delay
- flag R: if set, packet is not sensitive to reordering, and may be freely reordered [EDITOR'S NOTE: how does this interact with PSN/PSE?]
- ignored flags: reserved for use by overlying transport protocol
- PCF: see definition in {{path-communication-field}} 

## Measurement and Diagnosis using the Basic Header

[EDITOR'S NOTE: explain how to measure RTT and loss using CID/PSN/PSE. note that PSN increments on every packet.]

## On-Path State Maintenance using the Basic Header

[EDITOR'S NOTE: note rough TCP-equivalence of this state machine.]

~~~~~~~~~~~~~
    `- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -'
    `                        +============+                       '
    `                       /              \                      '
  +----------------------->(      zero      )<-+                  '
  ^ `                       \              /   |                  '
  | `                        +============+    |                  '
  | `                        a->b  |           | TO_IDLE          '
  | `                              V           |                  '
  | `                        +============+    |                  '
  | `                    +->/              \   |                  '
  | `               a->b | (    uniflow     )--+                  '
  | `                    +--\              /                      '
  | `                        +============+                       '
  | `- - - - - - - - - - - - - - - | b->a  - - - - - - - - - - - -'
  |                                V
  | TO_IDLE                  +============+  
  +<------------------------/              \ 
  |                        (  associating   )
  |                         \              / 
  | TO_ASSOCIATED            +============+  
  +<-----------------------+        | a->b
  |                         \       V       
  |                          +============+ 
  |                      +->/              \
  |                a<->b | (   associated   )
  |                      +--\              /
  | TO_ASSOCIATED            +============+ 
  +<-----------------+        | stop y->z   
  |                   \       V             
  |                    +============+  
  |                +->/              \ 
  |      any a<->b | (    stopping    )
  |                +--\              /      
  | TO_STOPWAIT        +============+       
  +------------+        | stop z->y
                \       V
                 +============+
                /              \
               (   stop-wait    )
                \              /
                 +============+
~~~~~~~~~~~~~
{: #fig-states title="Transport-independent state machine as implemented by PLUS"}


## Bidirectional Stop Signaling

[EDITOR'S NOTE: describe bidirectional stop signaling and explain why it works.]

# Path Communication Field

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
+-+-+-+-+------+---+-+-+-------+-------------------------------+
|S|0|L|R| ign  |rsv|D|0|   T   |            PCF value          |
+-+-+-+-+------+---+-+-+-------+-------------------------------+
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
+-+-+-+-+------+---+-+-+-+-----+-------------------------------+
|S|0|L|R| ign  |rsv|D|1|0|  T  |          PCF value            |
+-+-+-+-+------+---+-+-+-+-----+                               |
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
+-+-+-+-+------+---+-+-+-+-+---+---------------+---------------+
|S|0|L|R| ign  |rsv|D|1|1|0| T |  PCF length   |               /
+-+-+-+-+------+---+-+-+-+-+---+---------------+               \
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

## Two-byte Path to Receiver signals 

(Codepoints to be assigned)

- Path MTU accumulator. Two bytes of value containing the accumulated path MTU, measured in bytes, initialized to the sender's MTU by the sender. A PLUS-aware forwarding device on path receiving this value MUST fill the minimum of the recieved value and the MTU of the next hop into this field.
- Path state timeout accumulator. Two bytes of value, unsigned 16-bit integer timeout in seconds, initialized to 65535 (max timeout). A PLUS-aware state-keeping device on path that will time out state MUST fill the minimum of the received value and its current timeout in seconds into this field.
- Path rate intent accumulator. (Define how this works: logarithmic scaled value for bandwidth demand?) 

# Six-byte Path to Receiver Signals

(Codepoints to be assigned)

- Path trace accumulator, similar to the Path Changes mechanism in section 4.3 of {{IPIM}}.

## Two-byte Sender to Path signals

(Codepoints to be assigned)

- Sender rate intent. Two bytes of value. (Define how this works: logarithmic scaled value for bandwidth demand?)

## Six-byte Sender to Path signals

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
