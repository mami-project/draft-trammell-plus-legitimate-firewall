---
title: Transport-Independent Path Layer State Management
abbrev: PLUS Statefulness
docname: draft-trammell-plus-statefulness
date: 2016-09-12
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
  RFC2474:
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

TO WRITE: stateful on-path devices exist, they try to model TCP, whether completely or
incompletely, this is bad because not everything is TCP. encrypted transports
can expose basic information about their operation, on-path devices can use this information to provide tcp-equivalent state modeling. necessary for transports over UDP like QUIC because otherwise you're stuck in UDP timeout hell.

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

The transport-independent state machine for on-path devices is shown in {{fig-
states}}. It relies on four states, three configurable timeouts, and a set of signals defined in {{abstract-signaling-mechanisms}}. The states are defined as follows:

- zero: there is no state for a given flow at the device
- uniflow: a packet has been seen in one direction (optionally, the initiator-to-responder direction; see {{initiator-signaling}}); state will be kept at the device until it is explicitly cancelled or until timeout t1 elapses without a packet.
- biflow: a packet has been seen in one direction and an indication of consent has been seen in the opposite direction; state will be kept at the device until it is explicitly cancelled or until timeout t2 elapses without a packet.
- closing: an established biflow is shutting down due to an explicit close indication; state will be kept at the device until timeout t3 elapses.

t1 is the non-directional idle timeout. It can be considered equivalent to the
idle timeout for transport protocols where the device has no information about
session start and end (e.g. most UDP protocols). t2 is the bidirectional idle
timeout. It can be considered equivalent to the timeout for transport
protocols where the device has information about session start and end (e.g.
TCP). t3 is the shutdown timeout: how long the device will wait for reordered
packets after a shutdown signal. Selection of timeouts is a 

~~~~~~~~~~~~~
 _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
'           (initiator)             ' 
'  +--------+ packet  +----------+  '  UNIFLOW
'  |  zero  |-------->| uniflow  |  '  ONLY 
'  |        |<--------|          |  '  STATES
'  +--------+ t1/stop +----------+  '
'_ _ ^ _ _ ^ _ _ _ _ _ _ _ | _ _ _ _'
   t3|      \___________   | consent
     |           t2     \  v packet    ADDITIONAL
   +---------+        +----------+     BIFLOW
   | closing |<-------|  biflow  |     STATES
   |         |  stop  |          |
   +---------+        +----------+
    
~~~~~~~~~~~~~
{: #fig-states title="Transport-Independent State Machine for Stateful On-Path Devices"}

[TODO: edit to fit changes to diagram: When a stateful network devices receives the first packet of a flow that is
does not have state for (in "no state" state), it will transits in the "flow"
state and set a time t1. Every time a new packet arrives that belongs to the
same uniflow, t1 is reset. Only if a packet is recieved that is identified to
belong to the associated reverse flow, the network device will transition into
"biflow" state and use a larger time t2 instead. If an explicit stop signal is
received from one of the bi-flows (from any of the endpoints), this indicates
that the network device can remove state information for this biflow. If the
network device can determine that all sent packets have successfully
traversed, e.g. based on sequence numbers, it can directly move to the "no
state" state and remove all state information on this flow. Otherwise it
should move into a "closing" state and again set a shorter timer t1 (could
even be t3>t1) for each received packet. Again, if additional information such
as a sequence number is available to determine if all packets have traversed
the network device, it might also directly go over in the "no state" state
from "closing" after the last packet that e.g. could have been delayed to
after the stop signal due to re-ordering. Otherwise, whenever a timer expire,s
the network device will remove all state information for this flow and move to
the "no state" state.

This state machine is especially considered for stateful network devices that
will drop packet in the "flow" state. Other network functions that do not
require biflow information, might only implement the "no state" and "flow"
states and only use a short timer that will be refreshed for all packets
received. Depending on the timer configuration these devices might utilize the
stop signal or not.]

# Abstract Signaling Mechanisms

[WRITE THIS: note there are a few ways to implement this, we describe a few independent mechanisms that may be combined. concrete mechanism up to a future protocol specification. note that any signaling bits need to be integrity protected.]

## Consent Signaling

[WRITE THIS: consent flow must
at least be medium-assurance of implementation and on-path-ness. consent token
is derived from forward token.]

## Stop Bit Signaling

[WRITE THIS: one bit, integrity-protected, to say stop]

## Initiator Signaling

[WRITE THIS: additional optional bit to say "i'm the flow initiator", on every packet. this might make some firewalls easier to implement...?]

## Header-Only Forwarding

[TODO: edit: A network device in "no state" might only forward the unencrypted/header parts of a packet to transition to "biflow" state without forwarding unknow data. A PLUS receiver MUST reply with a (PLUS-only) packet (within a given time frame). On receiption of this PLUS-only packet, the network device tranits in "biflow" state and SHOULD resend the whole packet including the encrypted data while dropping a PLUS-only packet. If the network device was not able to store the encrypted data, it forwards the PLUS-only packet to the original sending endpoint which then can be used as a fast trigger that the original encrypted data was not received by the intented receiving endpoint.]

## Timeout Exposure

[WRITE ME: additional information: use {{I-D.trammell-plus-abstract-mech}} to expose the time-outs used to the endpoints]

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