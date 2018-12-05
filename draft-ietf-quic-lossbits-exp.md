---
title: The QUIC Loss Bits
abbrev: QUIC Loss Bits
docname: draft-ietf-quic-lossbits-exp-latest
date: {DATE}
category: exp

ipr: trust200902
workgroup: QUIC
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: A. Ferrieux
    role: editor
    name: Alexandre Ferrieux
    org: Orange Labs
    email: alexandre.ferrieux@orange.com
  -
    ins: I. Hamchaoui
    role: editor
    name: Isabelle Hamchaoui
    org: Orange Labs
    email: isabelle.hamchaoui@orange.com

normative:

  QUIC-TRANSPORT:
    title: "QUIC: A UDP-Based Multiplexed and Secure Transport"
    date: {DATE}
    seriesinfo:
      Internet-Draft: draft-ietf-quic-transport-latest
    author:
      -
        ins: J. Iyengar
        name: Jana Iyengar
        org: Fastly
        role: editor
      -
        ins: M. Thomson
        name: Martin Thomson
        org: Mozilla
        role: editor

 QUIC-RECOVERY:
title: "QUIC Loss Detection and Congestion Control"
abbrev: QUIC Loss Detection 
docname: draft-ietf-quic-recovery-latest 
date: {DATE} 
category: std ipr: trust200902 area: Transport workgroup: QUIC

stand_alone: yes pi: [toc, sortrefs, symrefs, docmapping]

author:
ins: J. Iyengar
name: Jana Iyengar
org: Fastly
email: jri.ietf@gmail.com
role: editor
ins: I. Swett name: Ian Swett org: Google email: ianswett@google.com role: editor





--- abstract

This document specifies the addition of loss bits to the QUIC
transport protocol and describes how to use them to measure and locate packet loss.
--- note_Note_to_Readers

This document specifies an experimental delta to the QUIC transport protocol. 
--- middle
# Introduction

Packet  loss is  a hard  and  pervasive problem  of day-to-day  network
operation,  and  locating them  is  crucial  to timely  resolution  of
crippling  end-to-end  throughput  issues.    To  this  effect,  in  a
TCP-dominated world,  network operators  have been heavily  relying on
information   present  in   clear   in  TCP   headers:  sequence   and
acknowledgement  numbers, and  SACK when  enabled. By  passive on-path
observation,  these  allow  for   quantitative  estimation  of  packet
loss. Additionally, the lossy segment (upstream or downstream from the
observation point)  is unambiguous;  this is crucial  as it  gives the
ability to  quickly home in  on the  offending segment, by  moving the
passive observer around.


In the QUIC context, the equivalent transport headers being encrypted,
such observation is not possible. To restore network operators' ability
to maintain QUIC clients experience, this document  adds two explicit loss bits to the QUIC  short header,
named "Q" and "R". Together, these bits allow the observer to estimate
upstream and downstream  loss, enabling the same  dichotomic search as
with TCP.


# Passive Loss measurement

The proposed mechanisms enable loss measurement from observation points on the network path throughout the lifetime of a connection. End-to end loss as well as segmental loss (upstream or downstream from the observation point) are measurable thanks to two dedicated bits in short packet headers, named loss bits. The loss bits therefore appear only after version negotiation and connection establishment are completed.

## Proposed Short Header Format Including Loss Bits

As of the current editor's version of {{QUIC-TRANSPORT}}, this proposal
specifies using the seventh amost significant bit (0x02) of the first byte in
the short header for the sQuare bit and the eight amost significant bit (0x01) of the first byte in
the short header for the Retransmit bit. The sQuare and Retransmit bits constitute the Loss bits.

~~~~~

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+
|0|K|1|1|0|S|Q|R|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                Destination Connection ID (0..144)           ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Packet Number (8/16/32)                ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Protected Payload (*)                   ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

~~~~~
{: #fig-short-header title="Short Header Format including proposed Q and R Bit"}

Q: The sQuare bit is toggled every 64 outgoing packets as explained in {{squarebit}}.

R: The Retransmit bit is set to 0 or 1 according to the not-yet-disclosed-lost-packets
counter, as explained in {{retransmitbit}}.

## Setting the Square Bit on Outgoing Packets {#squarebit}

Each endpoint independently maintains  a sQuare value, 0 or 1, during a block of N outgoing packets (e.g. N=64), and sets the sQuare  bit in the short header to the currently stored value when a packet with a short header is sent out. The sQuare value is initiated to 0 at each endpoint, client and server, at connection start.
This mechanism delineates thus slots of N packets with the same marking. Observation points can estimate the  upstream losses  by simply counting the number of packets during an half period of the square signal, as described in {{usage}}.


## Setting the Retransmit Bit on Outgoing Packets {#retransmitbit}

Each endpoint, client and server, independently maintains a not-yet-disclosed-lost-packets counter and sets the Retransmit bit of short header packets to 0 or 1 accordingly.
The not-yet-disclosed-lost-packets counter is initialized to 0 at each endpoint, client and server, at connection start, and reflects packets considered lost by the QUIC machinery which content is pending for retranmission. When a packet is declared lost by the QUIC transmission machinery (see https://github.com/quicwg/base-drafts/blob/ac5e4af758cd61329244297737b93c87c3889e3d/draft-ietf-quic-recovery.md#loss-detection) the not-yet-disclosed-lost-packets counter is incremented by 1. When a packet with a short header is sent out by an end-point, its retransmit bit is set at 0 when the not-yet-disclosed-lost-packets counter is equal to 0. Otherwise, the packet is sent out with a retransmit bit set to 1 and the not-yet-disclosed-lost-packets counter is decremented by 1.

Observation points can estimate the number of packets considered lost by the QUIC transmission machinery in a given direction by observing the number of packets which retransmit bit is set to 1 in this direction. This estimation of end-to-end loss is delayed at least by the uplink one way delay (in case of fast retransmit). 


{{?QUIC-SPIN=I-D.trammell-quic-spin}} 

## Resetting sQuare Value 

Each client and server resets its sQuare value to zero and initiates a new slot of 64 packets when sending the first
packet of a given connection with a new connection ID. This reduces the risk that transient sQuare bit state can be used to link flows across connection migration or ID change.


## Resetting Retransmit Value 
The not-yet-disclosed-lost-packets counter is reset at each endpoint, client and server, when sending the first packet of a given connection with a new connection ID. This reduces the risk that transient Retransmit bit state can be used to link flows across connection migration or ID change.

# Using the loss bits for Passive Loss Measurement {#usage}

## End-to-end loss 
The Retransmit bit mechanism merely reflects the number of packets considered lost by the sender QUIC stack with a slight delay. In case of fast retransmit due to repeted acknowlegments of a packet, this delay is at least equal to the uplink one way delay. It is larger otherwise. The retransmit mechanism alone suffices to estimate the end-to-end losses; similar to TCP passive loss measurement, its accuracy depends on the loss affecting the retransmitt-bit-marked packets, which are in themselves proof of previous loss.
## Upstream loss 
During a QUIC connection lifetime, the sQuare bit mechanism delineates slots of N packets with the same marking. When focusing on the sQuare bit of consecutive packets in a direction, this mechanism sketches a periodic sQuare signal which toggles every N packets. On-path observers can then estimate the  upstream losses by simply counting the number of packets during a half period (level 0 or level 1) of the square signal.
Packets with a long header are not marked, but yet taken into account by the sender when counting the N outgoing packets before its next toggle. Observers should assign long header packets to the pending slot if possible (i.e. up to N packets counted in this slot), to the next one otherwise. Thus, slots with less than N packets, whatever their header length, generally denote upstream loss. 
As with TCP passive detection based on missing sequence numbers, this estimation may become incaccurate in case of packet reordering which blurs the edges of the square signal ; heuristics may be proposed to filter out this noise in the observation points. 

The slot size N should be carefully chosen : too short, it becomes very sensitive to packet reordering and loss. Too large, short connections may end before completion of the first square slot, preventing any loss estimation. Slots of 64 packets are suggested.

## Downstream loss 

 The retransmit bit mechanism may be coupled with the square bit mechanism to estimate downstream losses. Indeed, passive observers can infer downstream losses by difference between end-to-end and upstream losses. 
 The sQuare bit mechanism allows for observers to compute loss measurment at the end of every half sQuare signal period (level 0 or level 1).
 The retransmit bit mechanism provides for the end-to-end loss after reaction of the sender stack.
 
 On path observers can estimate upstream and downstream loss at various scales, from the N slot level to the connection lifetime level.
 
 
 Note that observers should perform a loose synchronisation between the sQuare and the retransmit measurements when accurate evolution of segmental loss over connection lifetime is sought so as to compare the same portion of the packet stream.
 
 
 
# IANA Considerations

This document has no actions for IANA.

# Security and Privacy Considerations

The loss bits are intended to expose loss to observers along the path, so the privacy considerations for the loss bits are essentially the same as those for passive loss measurement in general. Loss gives no hint on customer geolocalisation; moreover, reset of the loss counting at conenction Id change prevent for linkability.


# Change Log

> **RFC Editor's Note:**  Please remove this section prior to
> publication of a final version of this document.



# Acknowledgments
{:numbered="false"}
 
The sQuare Bit was originally specified by Kazuho Oku for RTT measurment.



