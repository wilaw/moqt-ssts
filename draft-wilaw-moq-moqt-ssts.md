---
title: "Sender-Side Track Switching"
abbrev: "SSTS"
category: std

docname: draft-wilaw-moq-moqt-ssts-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - moq
 - moqt
 - abr
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"

author:
  - fullname: Will Law
    organization: Akamai
    email: "wilaw@akamai.com"
  - name: Ian Swett
    organization: Google
    email: ianswett@google.com
  - name: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com
  - name: Mo Zanaty
    organization: Cisco
    email: mzanaty@cisco.com
  - name: Victor Vasiliev
    organization: Google
    email: vasilvv@google.com
  - name: Ali Begen
    organization: Ozyegin University
    email: ali.begen@ozyegin.edu.tr
  - name: Zafer Gurel
    organization: Ozyegin University
    email: zafer.gurel@ozu.edu.tr
  - name: Gwendal Simon
    organization: Synamedia
    email: gsimon@synamedia.com

normative:
  MOQT: I-D.draft-ietf-moq-transport-19

informative:

--- abstract

This draft defines an extension to MOQT to enable Sender-Side Track Switching.


--- middle

# Introduction

Sender Side Track Switching (SSTS) is a subscriber-initiated and controlled behavior in which
a publisher dynamically selects which track to forward from a switching set based on
various algorithms. Each algorithm defines a set of attributes which are passed in the
SWITCHING-SET-ASSIGNMENT parameter {{switching-set-assignment-param}} along with a
complimentary set of rules for subscriber behavior and relay behavior.

This specification defines a default algorithm - type 0. Other algorithms are referenced in the
"SSTS Algorithms" registry {{iana-ssts-algorithms}}.

SSTS is implemented as a MOQT Extension(See {{MOQT}} Sect 3.2).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Extension negotiation

During MOQT SETUP, an endpoint communicates which SSTS algorithms it supports by passing the
SSTS_ALGORITHMS Setup Option {{ssts-algorithms}}.

The absence of the SSTS_ALGORITHMS setup option, or an SSTS_ALGORITHMS setup option
with an empty list, prohibits the use of SSTS.

## SSTS_ALGORITHMS {#ssts-algorithms}
The SSTS_ALGORITHMS option (Option Type 0x09) communicates the list of SSTS
algorithms which the relay supports. Supported algorithms are serialized as
a sequence of varints. Returning an empty sequence is acceptable and indicates
that SSTS is not supported. Algorithms are registered in the SSTS-Algorithms
{{iana-ssts-algorithms}} registry.

# General behaviors for all SSTS algorithms {#ssts-general-requirements}

The subscriber is responsible for grouping tracks into switching sets based on application-level
knowledge. A switching set is a collection of tracks representing the same content encoded at
different throughput levels, typically from a single source. Tracks within a switching set are
time-aligned at certain group boundaries, allowing the relay to switch between tracks at these
boundaries while ensuring the subscriber receives uninterrupted content from the set. The relay
selects exactly one track per switching set to forward at any given time.

Subscribers can create switching sets through three methods. All support single or multiple
switching sets and result in identical relay behavior:

* Individual SUBSCRIBE: - the subscriber sends a separate SUBSCRIBE message for each track,
  and appends the SWITCHING-SET-ASSIGNMENT parameter to assign the track to a switching set.

* SUBSCRIBE_TRACKS: - the subscriber sends a SUBSCRIBE_TRACKS message. For each matching
  track, the relay will issue a PUBLISH message. The subscriber assigns tracks to switching sets
  by appending the SWITCHING-SET-ASSIGNMENT parameter to the PUBLISH_OK message.

* PUBLISH: - the publisher sends a PUBLISH message. The subscriber assigns tracks to switching sets
  by appending the SWITCHING-SET-ASSIGNMENT parameter to the PUBLISH_OK message.

In all cases, tracks are grouped into a switching set by specifying the same switching set ID.

# SWITCHING_SET_ASSIGNMENT Parameter {#switching-set-assignment-param}

The SWITCHING-SET-ASSIGNMENT parameter (Parameter Type 0x41) MAY appear in a SUBSCRIBE,
REQUEST_UPDATE, or PUBLISH_OK message. This parameter assigns a subscription to a SSTS
switching set and specifies the algorithm to be used for switching. Each algorithm MAY
extend the serialization to pass additional fields.

~~~
SWITCHING-SET-ASSIGNMENT {
  Switching set ID (vi64),
  Algorithm (vi64)
}
~~~

* Switching set ID: Integer identifying the switching set. A track MUST only be assigned
  to one switching set at a time. If a subscription attempts to assign a track that is
  already assigned to a different switching set, the relay MUST reject the subscription
  with a Parameter Error.
* Algorithm: integer identifying the SSTS algorithm to be used.


# Default switching algorithm

This specification defines a default SSTS algorithm with a type of 0.

## SWITCHING-SET-ASSIGNMENT fields
This algorithm extends the base definition of the SWITCHING-SET-ASSIGNMENT parameter
{{switching-set-assignment-param}} to add the following fields:

SWITCHING-SET-ASSIGNMENT {
  Switching set ID (vi64),
  Algorithm ID (vi64),
  Throughput threshold (vi64),
  Set throughput weight (vi64),
  Activate switching (vi64),
  Set rank (8)
}

* Throughput threshold: Minimum throughput (kbps) required to select this track.

* Set throughput weight: Relative weight for bandwidth allocation, expressed as an
  integer 1 <= N <= 10. Each set receives bandwidth proportional to its weight:
  `target = B_total × weight / sum_F`. These are relative weights, not absolute
  percentages; for example, weights of 6, 4, 3 (sum = 13) allocate 46%, 31%, 23%
  respectively. This allows sets to be added or removed without requiring other sets to
  update their weights. When multiple subscriptions in the same switching set specify
  different weight values, the publisher MUST use the value from the most recently received
  message for that set.

* Activate switching: Integer, when set to 0, pauses SSTS switching for this set. When set
  to N, the relay activates or resumes switching as soon as the number of tracks assigned to
  the switching set is >= N.  Activation takes effect when an Object is received or published
  on a Group larger than previously largest Group. When multiple subscriptions in the same
  switching set specify different activate values, the publisher MUST use the value from the
  most recently received message for that set.

* Set rank: Degradation priority when bandwidth is constrained, expressed as an 8-bit unsigned
  integer (0-255). Default is 0. Lower values indicate higher priority (protected from degradation).
  See {{allocation-algorithm}} for details. When multiple subscriptions in the same switching set
  specify different rank values, the publisher MUST use the value from the most recently received
  message for that set.

## Subscriber behavior

The subscriber follows the general rules {#ssts-general-requirements} for switching set establishment.

The subscriber sets activate switching = N, where N is the number of tracks that will be assigned to that
switching set.

To modify an established switching set, the subscriber can

* Add a track to an existing set: send SUBSCRIBE or PUBLISH_OK with a SWITCHING-SET-ASSIGNMENT parameter
  referencing an existing set.
* Remove a track from a set: unsubscribe from that track.
* Pause SSTS: Send REQUEST_UPDATE for any track assigned to that set with a SWITCHING-SET-ASSIGNMENT
  parameter defining activate = 0.
* Resume SSTS: Send REQUEST_UPDATE for any track assigned to that set with a SWITCHING-SET-ASSIGNMENT
  parameter defining activate switching = N, where N is the number of tracks assigned to that switching set.

## Publisher behavior

When the publisher receives a subscription with SWITCHING-SET-ASSIGNMENT:

1. Add the subscription to the specified switching set, creating the set if needed.
2. Set Forward state to 0 for the new subscription, irrespective of the forward state received from the
   SUBSCRIBE or PUBLISH_OK.
4. Store 'throughput threshold' as a property of the subscription.
5. Store 'Set throughput weight', 'Set rank' and 'Activate switching' as properties of the set.
6. If the number of tracks assigned to the set with active subscriptions >= the activate switching value,
   then begin active track selection by applying the bandwidth allocation algorithm {{allocation-algorithm}}
   when an Object is received or published on a Group larger than previously largest Group.

If the publisher receives a PUBLISH_DONE message, or an UNSUBSCRIBE for a subscription that was
previously added to a switching set, then it must remove that subscription from the switching set
and continue to process the switching across the remaining subscriptions within that set. The value of
'activate switching' MUST be decremented by one to enable the swictching to remain active.

If all tracks are removed from a previously established switching set, then that set is
considered deleted and is removed from the bandwidth allocation algorithm.

Publishers SHOULD maintain a forward=1 upstream state (if any) on all tracks within a switching set,
irrespective of their downstream forwarding state.

### Bandwidth Allocation {#allocation-algorithm}

The publisher maintains:

- `B_total`: Estimated downstream bandwidth capacity for the subscriber connection. The
  publisher maintains a bandwidth estimate for each downstream subscriber. The timebase of this
  estimate SHOULD be at least the Group duration of the track, if that is known or can be estimated
  by the publisher, or several seconds if it is unknown. The estimate is obtained
  periodically from the transport stack (e.g., congestion window pacing rate, smoothed RTT)
  and MAY be supplemented by external sources or application-level feedback. The exact mechanism
  is not defined by this algorithm and might vary between implementations.
- `sum_W`: Sum of all set weights updated incrementally as subscriptions are added or removed
- 'set.weight': for each switching set, the switching set weight, as defined by the set
  throughput weight of the SWITCHING-SET-ASSIGNMENT {{switching-set-assignment-param}} parameter.
- 'set.rank': for each switching set, the switching set rank, as defined by the set
  rank field of the SWITCHING-SET-ASSIGNMENT {{switching-set-assignment-param}} parameter.

On a periodic update interval or at a minimum when an object is received/published on a group
larger than previously largest group, the relay executes the following algorithm:

~~~
B_remaining  = B_total
for each set in ascending set.rank order:
  set.target = B_remaining × set.weight / sum_W
  set.allocated = min(set.target, B_remaining)
  set.selected = track in set with highest throughput_threshold where track.throughput_threshold <= set.allocated
  B_remaining -= set.selected.throughput_threshold
for each track in a switching set:
  set forward state = (track == set.selected)
~~~

The rank ordering ensures higher-priority sets receive their target allocation first; lower-priority sets
absorb any bandwidth shortfall. When bandwidth is sufficient, all sets receive `allocated = target`.
When bandwidth is constrained, higher-priority sets (lower rank value) are protected while
lower-priority sets receive less than their target or nothing.


# Security Considerations

TBD

# IANA Considerations

## SSTS_ALGORITHMS Setup Option

IANA is requested to add the following entry to the "Setup Options"
registry (Section 15.4 of {{MOQT}}):

| Type | Name                   | Specification  |
|------|------------------------|----------------|
| TBD1 | SSTS_ALGORITHMS  | This document  |

SSTS_ALGORITHMS is a Setup Option (see {{ssts-algorithms}})
that an endpoint includes in its SETUP message to indicate support for the
SSTS extension defined in this document.

## SSTS-Algorithms {#iana-ssts-algorithms}

This document establishes a registry for SSTS algorithms. The
registration policy is Specification Required (per {{!RFC8126,
Section 4.6}}).

| Type | Name       | Specification |
|-----:|:-----------|:--------------|
| 0x0  | Default  | this |




TODO acknowledge.
