---
title: Transport-Independent Firewall State Management using PLUS
abbrev: PLUS Firewall Support
docname: draft-trammell-plus-legitimate-firewall
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

--- abstract

This document currently describes a simple state machine for stateful network devices. This state machine is used to determine the input signals that would need to be provided by an protocol that assotiates packets to a common flow.

--- middle

# Introduction

# Terminology

# State Machine

~~~~~~~~~~~~~
 _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
'   ---------   pkt   ---------   '
'   |  no   |-------->| flow  |   '
'   | state |<--------|       |   '
'_ _---------_t1/stop_---------_ _'
     ^    ^                |
   t1|    |___________     | reverse pkt
     |       t2/stop  |    v
   -----------       ------------
   | closing |<------|  biflow  |
   -----------  stop ------------
    
~~~~~~~~~~~~~

When a stateful network devices receives the first packet of a flow that is does not have state for (in "no state" state), it will transits in the "flow" state and set a time t1. Every time a new packet arrives that belongs to the same uniflow, t1 is reset. Only if a packet is recieved that is identified to belong to the associated reverse flow, the network device will transition into "biflow" state and use a larger time t2 instead. If an explicit stop signal is received from one of the bi-flows (from any of the endpoints), this indicates that the network device can remove state information for this biflow. If the network device can determine that all sent packets have successfully traversed, e.g. based on sequence numbers, it can directly move to the "no state" state and remove all state information on this flow. Otherwise it should move into a "closing" state and again set a shorter timer t1 (could even be t3>t1) for each received packet. Again, if additional information such as a sequence number is available to determine if all packets have traversed the network device, it might also directly go over in the "no state" state from "closing" after the last packet that e.g. could have been delayed to after the stop signal due to re-ordering. Otherwise, whenever a timer expire,s the network device will remove all state information for this flow and move to the "no state" state. 

This state machine is especially considered for stateful network devices that will drop packet in the "flow" state. Other network functions that do not require biflow information, might only implement the "no state" and "flow" states and only use a short timer that will be refreshed for all packets received. Depending on the timer configuration these devices might utilize the stop signal or not.

# Abstract Signaling Mechanisms

## Stop Bit

our thing based on Christian's

## Initiator/Responder

Joe's thing

## Header-Only Forwarding

A network device in "no state" might only forward the unencrypted/header parts of a packet to transition to "biflow" state without forwarding unknow data. A PLUS receiver MUST reply with a (PLUS-only) packet (within a given time frame). On receiption of this PLUS-only packet, the network device tranits in "biflow" state and SHOULD resend the whole packet including the encrypted data while dropping a PLUS-only packet. If the network device was not able to store the encrypted data, it forwards the PLUS-only packet to the original sending endpoint which then can be used as a fast trigger that the original encrypted data was not received by the intented receiving endpoint.  

## Timeout Exposure

use {{I-D.trammell-plus-abstract-mech}} to expose the used time-outs to the endpoints

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