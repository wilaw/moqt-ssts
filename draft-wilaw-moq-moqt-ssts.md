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
  - name: Will Law
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

--- abstract

This draft defines an extension to MOQT to enable Sender-Side Track Switching.


--- middle

# Introduction

Sender Side Track Switching (SSTS) is a subscriber-initiated and controlled behavior in which
a publisher dynamically selects which track to forward from a switching set based on
various algorithms. Each algorithm defines a set of attributes which are passed in the
SWITCHING_SET_ASSIGNMENT parameter {{switching-set-assignment-param}} along with a
complimentary set of rules for subscriber behavior and publisher behavior.

This specification defines a default algorithm - type 0. Other algorithms are referenced in the
"SSTS Algorithms" registry ({{iana-ssts-algorithms}}).

SSTS is implemented as a MOQT Extension (See {{MOQT}} Sect 3.2).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Extension negotiation

During MOQT SETUP, an endpoint communicates which SSTS algorithms it supports by passing the
SSTS_ALGORITHMS Setup Option {{ssts-algorithms}}.

The absence of the SSTS_ALGORITHMS setup option, or an SSTS_ALGORITHMS setup option
with an empty list, prohibits the use of SSTS.

## SSTS_ALGORITHMS {#ssts-algorithms}
This specification defines a new MOQT Setup Option called SSTS_ALGORITHMS. The SSTS_ALGORITHMS
option (Option Type 0x09) communicates the list of SSTS algorithms which the endpoint supports.
Supported algorithms are serialized as a sequence of varints. Returning an empty sequence is
acceptable and indicates that SSTS is not supported. Algorithms are registered in the SSTS-Algorithms
registry (see {{iana-ssts-algorithms}}).

# General behaviors for all SSTS algorithms {#ssts-general-requirements}

The subscriber is responsible for grouping tracks into switching sets based on application-level
knowledge. A switching set is a collection of tracks representing the same content encoded at
different throughput levels, typically from a single source. Tracks within a switching set are
time-aligned at certain group boundaries, allowing the publisher to switch between tracks at these
boundaries while ensuring the subscriber receives uninterrupted content from the set. The publisher
selects exactly one track per switching set to forward at any given time.

Subscribers can create switching sets through three methods. All support single or multiple
switching sets and result in identical publisher behavior:

* Individual SUBSCRIBE: the subscriber sends a separate SUBSCRIBE message for each track,
  and appends the SWITCHING_SET_ASSIGNMENT parameter to assign the track to a switching set.

* SUBSCRIBE_TRACKS: the subscriber sends a SUBSCRIBE_TRACKS message. For each matching
  track, the publisher will issue a PUBLISH message. The subscriber assigns tracks to switching sets
  by appending the SWITCHING_SET_ASSIGNMENT parameter to the PUBLISH_OK message.

* PUBLISH: the publisher sends a PUBLISH message. The subscriber assigns tracks to switching sets
  by appending the SWITCHING_SET_ASSIGNMENT parameter to the PUBLISH_OK message.

In all cases, tracks are grouped into a switching set by specifying the same switching set ID.

# SWITCHING_SET_ASSIGNMENT Parameter {#switching-set-assignment-param}

This extension defines a new MOQT Parameter named SWITCHING_SET_ASSIGNMENT.

The SWITCHING_SET_ASSIGNMENT parameter (Parameter Type 0x41) MAY appear in a SUBSCRIBE,
REQUEST_UPDATE, or PUBLISH_OK message. This parameter assigns a subscription to a SSTS
switching set and specifies the algorithm to be used for switching. Each algorithm MAY
extend the serialization to pass additional fields.

~~~
SWITCHING_SET_ASSIGNMENT {
  Switching set ID (vi64),
  Algorithm ID (vi64)
}
~~~

* Switching set ID: Integer identifying the switching set. A track MUST only be assigned
  to one switching set at a time. If a subscription attempts to assign a track that is
  already assigned to a different switching set, the publisher MUST reject the subscription
  with a Parameter Error.
* Algorithm ID: integer identifying the SSTS algorithm to be used.


# Default switching algorithm

This specification defines a default SSTS algorithm with a type of 0.

## SWITCHING_SET_ASSIGNMENT fields
This algorithm extends the base definition of the SWITCHING_SET_ASSIGNMENT parameter
{{switching-set-assignment-param}} to add the following fields:

~~~
SWITCHING_SET_ASSIGNMENT {
  Switching set ID (vi64),
  Algorithm ID (vi64),
  Throughput threshold (vi64),
  Set throughput weight (vi64),
  Activate switching (vi64),
  Set rank (8)
}
~~~

* Throughput threshold: Minimum throughput (kbps) required to select this track.

* Set throughput weight: Relative weight for bandwidth allocation among switching sets
  that share the same 'set.rank' value, expressed as an integer 1 <= N <= 10. Sets
  sharing a rank divide the bandwidth available to that rank tier proportionally to
  their weight — e.g. weights of 6, 4, 3 (sum = 13) among three same-rank sets allocate
  46%, 31%, 23% of that tier's available bandwidth, respectively. Weight has no effect
  across different rank values; a higher-priority set (lower 'set.rank') is served
  ahead of a lower-priority set regardless of relative weight. See
  {{allocation-algorithm}} for details. When multiple subscriptions in the same
  switching set specify different weight values, the publisher MUST use the value from
  the most recently received message for that set.

* Activate switching: Integer, when set to 0, pauses SSTS switching for this set. When set
  to N, the publisher activates or resumes switching as soon as the number of tracks assigned to
  the switching set is >= N. Activation takes effect when an Object is received or published
  on a Group larger than previously largest Group. When multiple subscriptions in the same
  switching set specify different activate values, the publisher MUST use the value from the
  most recently received message for that set.

* Set rank: Degradation priority when bandwidth is constrained, expressed as an 8-bit unsigned
  integer (0-255). Default is 0. Lower values indicate higher priority (protected from degradation).
  See {{allocation-algorithm}} for details. When multiple subscriptions in the same switching set
  specify different rank values, the publisher MUST use the value from the most recently received
  message for that set.

## Subscriber behavior

The subscriber follows the general rules {{ssts-general-requirements}} for switching set establishment.

The subscriber sets activate switching = N, where N is the number of tracks that will be assigned to that
switching set.

To modify an established switching set, the subscriber can

* Add a track to an existing set: send SUBSCRIBE or PUBLISH_OK with a SWITCHING_SET_ASSIGNMENT parameter
  referencing an existing set.
* Remove a track from a set: unsubscribe from that track.
* Pause SSTS: Send REQUEST_UPDATE for any track assigned to that set with a SWITCHING_SET_ASSIGNMENT
  parameter defining activate = 0.
* Resume SSTS: Send REQUEST_UPDATE for any track assigned to that set with a SWITCHING_SET_ASSIGNMENT
  parameter defining activate switching = N, where N is the number of tracks assigned to that switching set.

## Publisher behavior

When the publisher receives a subscription with SWITCHING_SET_ASSIGNMENT:

1. Add the subscription to the specified switching set, creating the set if needed.
2. Set Forward state to 0 for the new subscription, irrespective of the forward state received from the
   SUBSCRIBE or PUBLISH_OK.
3. Store 'throughput threshold' as a property of the subscription.
4. Store 'Set throughput weight', 'Set rank' and 'Activate switching' as properties of the set.
5. If the number of tracks assigned to the set with active subscriptions >= the activate switching value,
   then begin active track selection by applying the bandwidth allocation algorithm {{allocation-algorithm}}
   when an Object is received or published on a Group larger than previously largest Group.

If the publisher receives a PUBLISH_DONE message, or an UNSUBSCRIBE for a subscription that was
previously added to a switching set, then it MUST remove that subscription from the switching set
and continue to process the switching across the remaining subscriptions within that set. The value of
'activate switching' MUST be decremented by one, while maintaining a floor of zero.

If all tracks are removed from a previously established switching set, then that set is
considered deleted and is removed from the bandwidth allocation algorithm.

Publishers SHOULD maintain a forward=1 upstream state (if any) on all tracks within a switching set,
irrespective of their downstream forwarding state.

### Bandwidth Allocation {#allocation-algorithm}

The publisher maintains:
- 'B_total': Estimated downstream bandwidth capacity for the subscriber connection in kbps. The
  publisher maintains a bandwidth estimate for each downstream subscriber. The timebase of this
  estimate SHOULD be at least the Group duration of the track, if that is known or can be estimated
  by the publisher, or several seconds if it is unknown. The estimate is obtained
  periodically from the transport stack (e.g., congestion window pacing rate, smoothed RTT)
  and MAY be supplemented by external sources or application-level feedback. The exact mechanism
  is not defined by this algorithm and might vary between implementations.
- 'set.weight': for each switching set, the switching set weight, as defined by the set
  throughput weight of the SWITCHING_SET_ASSIGNMENT {{switching-set-assignment-param}} parameter.
- 'set.rank': for each switching set, the switching set rank, as defined by the set
  rank field of the SWITCHING_SET_ASSIGNMENT {{switching-set-assignment-param}} parameter.

Switching sets are allocated bandwidth using strict priority: 'set.rank' establishes a total
order across switching sets, and a switching set MUST receive its full computed allocation
before any switching set of lower priority (a higher 'set.rank' value) receives any bandwidth.
'set.weight' only arbitrates between switching sets that share the same 'set.rank' value; it
has no effect across different rank values.

On a periodic update interval or at a minimum when an object is received/published on a group
larger than previously largest group, the publisher executes the following algorithm:

~~~
B_remaining = B_total
for each distinct set.rank value r present among active switching sets, in ascending order:
  active = { switching sets with set.rank == r }
  tier_pool = B_remaining
  repeat:
    sum_W_active = sum of set.weight for set in active
    for each set in active:
      set.target = tier_pool × set.weight / sum_W_active
      set.selected = null
      if ( set.target >= lowest track.throughput_threshold in set )
          set.selected = track in set with highest throughput_threshold
                         where track.throughput_threshold <= set.target
    saturated = { set in active : set.selected == highest-throughput_threshold
                                   track available in set }
    for each set in saturated:
      tier_pool -= (set.selected != null) ? set.selected.throughput_threshold : 0
      remove set from active
  until ( active is empty OR saturated is empty )
  for each set with set.rank == r:
    B_remaining -= (set.selected != null) ? set.selected.throughput_threshold : 0
for each track in a switching set:
  set forward state = (track == set.selected)
~~~

A switching set that is the only one at its rank receives up to the entirety of
'B_remaining' as its target, since 'sum_W_active' reduces to its own weight — this is what
allows a higher-priority set to consume as much of 'B_total' as it needs, independent of
weight. A switching set that shares a rank with others initially divides that tier's
opening bandwidth proportionally by weight.

If a switching set in a tier cannot use its full proportional share — either because it
has already selected the highest-bitrate track available to it, or because no track in
the set can make use of the additional bandwidth — the unused portion of its share is
reallocated among the remaining switching sets in the same tier, recomputed proportionally
to their weights, and this repeats until every switching set in the tier has either
selected its highest-available track or the tier's bandwidth is fully claimed. Only
bandwidth that no switching set in the tier can use at all carries forward, via
'B_remaining', to the next, lower-priority tier.

Each redistribution round removes at least one switching set from further consideration
within its tier, or terminates outright; the process therefore converges within at most
as many rounds as there are switching sets sharing that rank. Since a switching set
typically contains a small number of tracks, this adds negligible computational cost
compared to a single-pass allocation.

If the allocated throughput for a set is lower than the lowest throughput threshold of any
track in that set, then no data from that set is forwarded.

# Security Considerations
This document relies on the session security properties of {{MOQT}} and does
not modify MOQT's authentication or authorization model. The risks discussed
below concern resource exhaustion enabled by an already-authorized subscriber,
not confidentiality or integrity.

## Asymmetric Resource Exhaustion via Switching Set Fan-out
A subscriber can assign a large number of high-bitrate tracks to a single
switching set while consuming little or no downstream bandwidth. Because
publishers SHOULD maintain a forward=1 upstream state on all tracks within a
switching set irrespective of their downstream forwarding state (see
{{ssts-general-requirements}}), a publisher can be induced to fetch and cache
a large volume of upstream content while forwarding little or nothing to the
subscriber.

This is functionally equivalent to a subscriber issuing individual SUBSCRIBE
messages with Forward=0 for a large number of tracks, and is therefore a
pre-existing risk in MOQT rather than one introduced by this extension.
Switching sets do make the pattern easier to trigger, since a single
SWITCHING_SET_ASSIGNMENT parameter can fan out to many tracks that a
publisher must actively fetch upstream. As with the general MOQT case, this
SHOULD be mitigated by general-purpose protections at the publisher and any
intermediate nodes performing the switching, such as per-subscriber limits on
the number and aggregate bitrate of concurrently subscribed tracks, quotas on
upstream fetch and cache resources, and monitoring for subscribers whose
upstream resource consumption is disproportionate to their delivered
downstream rate. This document does not define new protocol mechanisms for
this purpose.

## Misrepresented Throughput Threshold
The Throughput Threshold field of the SWITCHING_SET_ASSIGNMENT parameter
({{switching-set-assignment-param}}) is supplied by the subscriber and is not
independently verified by the publisher. A subscriber can declare a
Throughput Threshold far in excess of a track's actual encoded bitrate (for
example, declaring 500Mbps for a track encoded at 16Mbps). Because the
bandwidth allocation algorithm ({{allocation-algorithm}}) only selects a
track once `set.target` meets or exceeds its declared threshold, an
inflated threshold ensures the track is effectively never selected for
downstream delivery.

Combined with the upstream forward=1 requirement described above, this
allows a subscriber to cause a publisher to continuously fetch and cache a
track's full upstream bitrate while never delivering any of it downstream.
This is functionally equivalent to a subscriber issuing a SUBSCRIBE with
Forward=0 for that track, and represents the same category of risk as the
attack described above, disguised as switching set membership rather than an
explicit non-forwarding subscription.

Since MOQT already permits Forward=0 subscriptions as a normal feature, this
document does not define new protocol-level defenses against a
misrepresented Throughput Threshold. Publishers MAY apply the general-purpose
resource protections described above, and MAY treat a declared Throughput
Threshold that is grossly inconsistent with a track's observed or announced
encoding bitrate as a signal for additional scrutiny of that subscriber.

# IANA Considerations

## SSTS_ALGORITHMS Setup Option

IANA is requested to add the following entry to the "Setup Options"
registry (Section 15.4 of {{MOQT}}):

| Type | Name                   | Specification  |
|------|------------------------|----------------|
| 0x09 | SSTS_ALGORITHMS  | This document  |

SSTS_ALGORITHMS is a Setup Option (see {{ssts-algorithms}}) that an
endpoint includes in its SETUP message to indicate support for the
SSTS extension defined in this document.

## SWITCHING_SET_ASSIGNMENT Parameter

IANA is requested to add the following entry to the "Message Parameters"
registry (Section 15.7 of {{MOQT}}):

| Parameter Type | Parameter Name  | Specification  |
|------|------------------------|----------------|
| 0x41 | SWITCHING_SET_ASSIGNMENT  | This document  |

SWITCHING_SET_ASSIGNMENT is a Parameter (see {{switching-set-assignment-param}})
that assigns a subscription to a switching set, defines the switching algorithm
to be used and passes optional parameters as required by the algorithm.

## SSTS-Algorithms {#iana-ssts-algorithms}

This document establishes a registry for SSTS algorithms. The
registration policy is Specification Required (per {{!RFC8126,
Section 4.6}}).

| Type | Name       | Specification |
|-----:|:-----------|:--------------|
| 0x0  | Default  | This document  |


# Acknowledgments

IETF moq working group.
