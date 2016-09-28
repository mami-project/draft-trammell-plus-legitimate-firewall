---
title: Transport-Independent Path Layer State Management
abbrev: PLUS Statefulness
docname: draft-trammell-plus-statefulness
date: 2016-09-28
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: M. Kuehlewind
    name: Mirja Kuehlewind
    org: ETH Zurich
    email: mirja.kuehlewind@tik.ee.ethz.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland
  -
    ins: B. Trammell
    name: Brian Trammell
    org: ETH Zurich
    email: ietf@trammell.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland


informative:
  I-D.hamilton-quic-transport-protocol:
  I-D.trammell-plus-abstract-mech:
  RFC2474:
  IMC-GATEWAYS:
    title: An experimental study of home gateway characteristics (Proc. ACM IMC 2010)
    author:
      -
        ins: S. Hatonen
      -
        ins: A. Nyrhinen
      -
        ins: L. Eggert
      -
        ins: S. Strowes
      -
        ins: P. Sarolahti
      - 
        ins: M. Kojo

normative:
  RFC5103:
  RFC7011:
  RFC7398:

--- abstract

This document describes a simple state machine for stateful network devices on
a path between two endpoints to associate state with traffic traversing them
on a per-flow basis, as well as abstract signaling mechanisms for driving the
state machine. This state machine is intended to replace the de-facto use of
the TCP state machine or degenerate forms thereof by stateful network devices
in a transport-independent way, while still allowing for fast state timeout
of non-established or undesirable flows.

--- middle

# Introduction

The boundary between the network and transport layers was originally defined
to be that between information used (and potentially modified) hop-by-hop, 
and that used end-to-end. End-to-end information in the transport layer
is associated with state at the endpoints, but processing of network-layer
information was assumed to be stateless.

The widespread deployment of stateful middleboxes in the Internet, such as
network address and port translators (NAPT), firewalls that model the TCP
state machine to distinguish packets belonging from desirable flows from
backscatter and random attack traffic, and devices which keep per-flow state
for reporting and monitoring purposes (e.g. IPFIX {{RFC7011}} Metering
Processes), has broken this assumption, and made it more difficult to deploy
non-TCP transport protocols in the Internet.

The deployment of new transport protocols encapsulated in UDP with encrypted
transport headers (such as QUIC {{I-D.hamilton-quic-transport-protocol}}) 
will present a challenge to the operation of these devices, and their ubquity
likewise threatens to impair the deployability of these protocols. There are
two main causes for this problem: first, stateful devices often use an
internal model of the TCP state machine to determine when TCP flows start and
end, allowing them to manage state for these flows; for UDP flows, they must
rely on timeouts. These timeouts are generally short relative to those for TCP
{{IMC- GATEWAYS}}, requiring UDP- encapsulated transports either to generate
unproductive keepalive traffic for long-lived sessions, or to tolerate
connectivity problems and the necessity of reconnection due to loss of on-path
state.

This document presents a solution to this problem by defining a state machine
that is transport independent to be implemented at per-flow state-keeping
middleboxes as a replacement for degenerate TCP state modeling. Middleboxes
implementing this state machine using signals from a common UDP encapsulation
layer can have equivalent necessary state information to that provided by TCP,
reducing the friction between middleboxes and these new transport protocols.

# Terminology

In this document, the term "flow" is defined to be compatible with the
definition given in {{RFC7011}}: A flow is defined as a set of packets passing
a device on the network during a certain time interval. All packets belonging
to a particular Flow have a set of common properties. Each property is defined
as the result of applying a function to the values of:

1. one or more network layer header fields (e.g., destination IP
   address) or transport layer header fields (e.g., destination port
   number) that the device has access to;
2. one or more characteristics of the packet itself (e.g., number
   of MPLS labels, etc.);
3. one or more of the fields derived from packet treatment at the device
   (e.g., next-hop IP address, the output interface, etc.).

A packet is defined as belonging to a flow if it completely satisfies all the
defined properties of the flow. In general, the set of properties on deployed
devices includes the source and destination IP address, the source and
destination transport layer port number, the transport protocol number. The
differentiated services field {{RFC2474}} may also be included in the set of
properties defining a flow.

A bidirectional flow or biflow is defined as compatible with {{RFC5103}}, by
joining the "forward direction" flow with the "reverse direction" flow,
derived by reversing the direction of directional fields (ports and IP
addresses). Biflows are only relevant at devices positioned so as to see all
the packets in both directions of the biflow, generally on the endpoint side
of the service demarcation point for either endpoint as defined in the
reference path given in {{RFC7398}}.

# State Machine

The transport-independent state machine for on-path devices is shown in 
{{fig-states}}. It relies on four states, three configurable timeouts, and a set of
signals defined in {{abstract-signaling-mechanisms}}. The states are defined
as follows:

- zero: there is no state for a given flow at the device
- uniflow: a packet has been seen in one direction; state will be kept at the
  device until it is explicitly cancelled or until timeout t1 elapses without
  a packet.
- biflow: a packet has been seen in one direction and an indication of consent
  has been seen in the opposite direction; state will be kept at the device
  until it is explicitly cancelled or until timeout t2 elapses without a
  packet.
- closing: an established biflow is shutting down due to an explicit close
  indication; state will be kept at the device until timeout t3 elapses.

t1 is the unidirectional idle timeout. It can be considered equivalent to the
idle timeout for transport protocols where the device has no information about
session start and end (e.g. most UDP protocols). t2 is the bidirectional idle
timeout. It can be considered equivalent to the timeout for transport
protocols where the device has information about session start and end (e.g.
TCP). t3 is the shutdown timeout: how long the device will wait for reordered
packets after a shutdown signal. Selection of timeouts is a configuration and
implementation detail, but generally t3 <= t1 < t2.

~~~~~~~~~~~~~
   _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
  '                                         ' 
  '    +----+   packet    +-----+           '  
  '   /      \---------->/       \----+     '  UNIFLOW
  '  (  zero  )         ( uniflow )  packet '  ONLY 
  '   \      /<----------\       /<---+     '  STATES
  '    +----+   t1,stop   +-----+           '
  '_ _ _ ^_^ _ _ _ _ _ _ _ _ | _ _ _ _ _ _ _'
      t3 |  \                | consent
         |   \__________     v 
       +-----+     t2   \  +----+
      /       \          \/      \----+    
     ( closing )<--------( biflow )  packet
      \       /    stop   \      /<---+    
       +-----+             +----+
    
~~~~~~~~~~~~~
{: #fig-states title="Transport-Independent State Machine for Stateful On-Path Devices"}

When a packet is received for a flow that the device has no state for, and it
is configured to forward the packet instead of dropping it, it moves that flow
from the zero state into the uniflow state and starts a timer t1. It
resets this timer for any additional packet it forwards in the same direction
as long as the flow remains in the uniflow state. When timer t1 expires, or
when a stop signal is observed, the device drops state for the flow and
performs any processing associated with doing so: tearing down NAT bindings,
closing associated firewall pinholes, exporting flow information, and so on.
Note that devices that see only a single direction of a flow only have these
two states and the transitions between them.

A uniflow becomes a biflow when the device observes a consent signal. A
consent packet is a packet sent in the opposite direction from the packet sent
with certain properties as defined in {{signaling-consent}}. After
transitioning to the biflow state, the device starts a timer t2. It resets
this timer for any packet it forwards in either direction. The biflow state
represents a fully established bidirectional communication. When timer t2
expires, the device assumes that the flow has shut down without signaling as
such, and drops state for the flow, performing any associated processing. When
a stop signal is observed in either direction, the flow transitions to the
closing state.

When a flow enters the closing state, it starts a timer t3. When this timer
expires, the device drops state for the flow, performing any associated processing.

Devices may augment the transitions in this state diagram depending on their
function. For example, a firewall that decides based on some information
beyond the signals used by this state machine to shut down a flow may
transition it directly to a blacklist state on shutdown. This document is
concerned only with states and transitions common to transport- and function-
independent state maintenance.

# Abstract Signaling Mechanisms

The state machine in {{state-machine}} requires three signals: a packet (the
first packet observed in a flow in the zero state), a consent signal (allowing
a device to verify that an endpoint wishes a bidirectional communication to be
established or to continue), and a stop signal (noting that an endpoint wishes
to stop a bidirectional communication). Additional related signals may also be
useful, depending on the function a device provides. There are a few different
ways to implement these signals; here, we explore the properties of some
potential implementations.

We make the following assumptions about these signals:

- At least the endpoints can verify the integrity of the signals exposed. 
- Endpoints and devices on path can probabilistically verify that a originator of a signal is on-path.

## Consent Signaling

[WRITE THIS: consent flow must
at least be medium-assurance of implementation and on-path-ness. consent token
is derived from forward token.]

## Stop Bit Signaling

[WRITE THIS: one bit, integrity-protected, to say stop. should be sent by one endpoint as the last packet. t3 serves to ]

## Initiator Signaling

[WRITE THIS: additional optional bit to say "i'm the flow initiator", on every packet. this might make some firewalls easier to implement...?]

## Header-Only Forwarding

[TODO: probably cut this, potentially move it to the plus protocol document: A network device in "no state" might only forward the unencrypted/header parts of a packet to transition to "biflow" state without forwarding unknow data. A PLUS receiver MUST reply with a (PLUS-only) packet (within a given time frame). On receiption of this PLUS-only packet, the network device tranits in "biflow" state and SHOULD resend the whole packet including the encrypted data while dropping a PLUS-only packet. If the network device was not able to store the encrypted data, it forwards the PLUS-only packet to the original sending endpoint which then can be used as a fast trigger that the original encrypted data was not received by the intented receiving endpoint.]

## Timeout Exposure

[WRITE ME: additional information: use {{I-D.trammell-plus-abstract-mech}} to expose the time-outs used to the endpoints]

# Signal mapping for current transport protocols

[WRITE ME: how to handle TCP flags? What about unknown transports? ]

# IANA Considerations

This document has no actions for IANA.

# Security Considerations

TODO

# Acknowledgments

Thanks to Joe Hildebrand and Christian Huitema for the discussions leading to this document.

This work is supported by the European Commission under Horizon 2020 grant
agreement no. 688421 Measurement and Architecture for a Middleboxed Internet
(MAMI), and by the Swiss State Secretariat for Education, Research, and
Innovation under contract no. 15.0268. This support does not imply
endorsement.