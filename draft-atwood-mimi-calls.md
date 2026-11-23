---
title: "Media Keys for Calls in More Instant Messaging Interoperability (MIMI)"
abbrev: "MIMI Call Media Keys"
category: std
docname: draft-atwood-mimi-calls-00
submissiontype: IETF
number:
date:
v: 3
ipr: trust200902
area: "Applications and Real-Time"
workgroup: "More Instant Messaging Interoperability"
venue:
  group: "More Instant Messaging Interoperability"
  type: "Working Group"
  mail: "mimi@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/mimi/"
  github: "MarkAtwood/ietf-mimi-calls"
keyword:
 - MIMI
 - MLS
 - calling
 - conferencing
 - SFrame
 - key derivation

author:
 -
    fullname: Mark Atwood
    organization: Review Commit, LLC
    city: Seattle
    region: WA
    country: US
    email: mark@reviewcommit.com

normative:
  RFC9420:
  I-D.ietf-mimi-protocol:
  I-D.ietf-mimi-content:
  I-D.ietf-mimi-room-policy:
  I-D.ietf-mls-extensions:

informative:
  RFC9180:
  RFC7942:
  I-D.ietf-sframe-enc:

--- abstract

A MIMI room is an MLS group, and MLS can give an application a secret
derived from that group. A call among the room's members could take its
media keys from there, which would make the call end-to-end encrypted
with no key agreement of its own. This document says how, and says why
the shape of the room decides whether it is safe: the secret MLS exports
is computable by every member of the group, so it confines a call's media
only when the group is the call. It defines a MIMI application component
for media keys, derives the media secret for a call whose room is its
own, and describes what an implementation does instead for a call among
only some of a room's members.

--- middle

# Introduction

The MIMI drafts have the pieces of an end-to-end encrypted call and do
not say that they go together.

{{I-D.ietf-mimi-content}}, Section 5.10 carries the endpoint: a body part
whose disposition is `session` holds "a description of a multimedia
session, or a URI used to join one". {{I-D.ietf-mimi-room-policy}},
Section 5 describes the room: "A room can be both `fixed_membership` and
`parent_dependent`, for example, for room used for a multi-media call of
clients in a Group DM". {{I-D.ietf-mls-extensions}}, Section 4.4 supplies
key material: a forward-secure exporter giving one secret per application
component. {{I-D.ietf-mimi-protocol}}, Sections 10.2 and 10.3 register
application components already, so the registry a media component would
live in exists.

No document joins them. Section 5.10 states that "further discussion of
calling and conferencing functionality is out-of-scope of this document",
and nothing else takes the subject up. An implementation can therefore
assemble a keyed call out of machinery that all exists, and two
implementations will not agree, because nothing says which component
keys the media or what feeds it.

This document says. It also makes an argument the pieces do not make for
themselves, in {{why}}: the exported secret belongs to the whole MLS
group, so whether it is the right key for a call depends on whether the
group is the call. That turns room-policy's note about a call room from
an example into a requirement.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

A *room* is a MIMI room ({{I-D.ietf-mimi-protocol}}), and the MLS group
({{RFC9420}}) associated with it. A *call* is a real-time media session
among some or all of a room's participants. A *participant* of a call is
a user whose clients are entitled to the call's media.

# Why the Group Secret Is Not Always the Call Secret {#why}

{{I-D.ietf-mls-extensions}}, Section 4.4 defines the safe exporter. For
an application component identified by a ComponentID, it gives

~~~
SafeExportSecret(ComponentID) = tree_node_[LeafIndex(ComponentID)]_secret
~~~

whose root is the `application_export_secret`, derived from the group's
`epoch_secret`. Two properties of that construction decide this document.

The first is welcome: it is forward-secure. Section 4.4 requires an
implementation to treat a component's exported secret as consumed and to
"delete any source key material according to the deletion schedule in
Section 9.2 of {{RFC9420}}". A secret recovered later does not expose an
earlier call.

The second is the constraint: **every member of the group can compute
it.** The function takes a ComponentID and nothing else. There is no
per-call context, so a component receives one secret per epoch, and any
further derivation an application performs on that secret is equally
computable by every member. Entitlement to a call's media therefore
cannot be established by deriving keys from the group secret. It can only
be established by the membership of the group that the secret comes from.

The consequence is simple and easy to get wrong. A call among three
participants of a twenty-participant room, keyed from that room's
exported secret, is readable by all twenty, subject only to their
obtaining the media. The seventeen who are not on the call are not
attackers in any interesting sense: they are members in good standing of
the group whose key schedule produced the key. No protocol error has
occurred. The call is simply not confined to the people on it.

So a call keyed this way MUST be a call whose room is its own, and
{{own-room}} specifies that case. {{subset}} describes what an
implementation does for a call that is not.

# A Call in Its Own Room {#own-room}

## The Room

A call MAY be given a room of its own. Such a room is a
parent-dependent room ({{I-D.ietf-mimi-room-policy}}, Section 5) whose
parent is the room the call was started from, and whose participants are
the call's participants. Giving the call its own room makes the MLS
group's membership and the call's entitled set the same thing, which
{{why}} shows is what confines the media.

A room used this way SHOULD also be `fixed_membership`, so that the
call's participant set is settled when the call starts, which is the
combination Section 5 of {{I-D.ietf-mimi-room-policy}} names.

> Note: as of {{I-D.ietf-mimi-room-policy}}-04, Section 5 requires a
> parent-dependent room's participants to be a *strict* subset of the
> parent's, which forbids a call whose participants are all of the
> parent's. That is the commonest call in a small room. This document
> assumes the rule is a subset rather than a strict subset.

## The Media Component

This document defines a MIMI application component, `media_keys`, whose
ComponentID is registered in {{iana}}. The component performs no
operation on the room's application data; it exists so that media key
material has a ComponentID of its own, and so that the key schedule
separates it from every other component by the domain separation
Section 4.1 of {{I-D.ietf-mls-extensions}} provides.

## Deriving the Media Secret

For each epoch of the call room's MLS group, a client derives

~~~
media_secret = SafeExportSecret(media_keys)
~~~

of the group's cipher suite hash length, and uses it as the input keying
material of the media layer's own key schedule. A client MUST NOT use the
`media_secret` directly as a media encryption key.

This document does not define the media layer's key schedule.
{{I-D.ietf-sframe-enc}} has one, and a deployment using another supplies
its own; in both cases `media_secret` is the shared input and the media
layer derives from it. The boundary is deliberate: what this document
specifies is where the shared secret comes from, and who can compute it.

## Rekeying

Nothing in this document rekeys a call, because the group does it.

Every change of the MLS group's epoch changes `media_secret`. Adding a
participant to the call is a Commit, and so is removing one, so the media
key changes at each. A client MUST derive a new `media_secret` on
entering a new epoch and MUST stop using the previous one for new media.

Two properties follow, and they are the reason to prefer this shape:

* A participant removed from the call loses access to subsequent media at
  once, because the epoch they were removed in is the last one whose
  secret they can compute. No media-layer rekeying protocol is needed to
  achieve it.

* A participant added to the call cannot compute the secret of any
  earlier epoch, so earlier media stays unreadable to them.

# A Call Among a Subset of a Room {#subset}

An implementation may want a call among some of a room's participants
without creating a room for it. {{why}} rules out the exported secret for
this case. An implementation MAY instead distribute a media secret of its
own choosing to the call's participants using the Safe HPKE interface of
{{I-D.ietf-mls-extensions}}, Section 4.2:

~~~
SafeEncryptWithLabel(PublicKey, media_keys, Label, Context, media_secret)
~~~

encrypted once to each participating client's HPKE public key from the
group's ratchet tree. The secret then reaches the participants and no
other member of the room.

This costs what {{own-room}} gets from MLS. The initiator must mint the
secret, must distribute it again whenever the participant set changes,
and must define what a client does when it receives a secret for a call
it is already in. Removing a participant from a call in particular
requires a fresh secret distributed to the remainder, which is a
rekeying protocol this document does not specify. An implementation that
wants a call to be as safe as its membership SHOULD give the call its own
room.

# Security Considerations

The whole of {{why}} is a security consideration, and is the reason this
document exists rather than a sentence in another one.

An implementation that keys a call from a room's exported secret without
giving the call its own room has built a call that every member of the
room can decrypt. This is not detectable by the call's participants and
produces no protocol error. It is the failure this document is written to
prevent.

A member of the room that hosts a call room is not thereby a participant
of the call: the call room is a different MLS group with a different key
schedule, and membership of the parent conveys no ability to compute the
child's secrets. The reverse is also true, which is worth stating because
the room-policy relationship might suggest otherwise: a participant of a
parent-dependent room computes nothing about its parent's traffic beyond
what its own membership of the parent gives it.

Media is carried outside MLS, typically through a media server that
forwards it. Such a server learns who is speaking to whom and for how
long whether or not it can read the media. This document does not change
that, and a deployment that needs to conceal it needs more than key
management.

The `media_secret` is key material. A client MUST NOT send it over MIMI,
MUST NOT log it, and MUST discard it when it leaves the call's group.

# IANA Considerations {#iana}

This document requests one entry in the MIMI App Component registry
established by {{I-D.ietf-mimi-protocol}}:

| Name | Value | Description | Reference |
|---|---|---|---|
| media_keys | TBD | Media key material for a call | RFCXXXX |

Implementations before assignment use a provisional value agreed between
them, and such a value is not interoperable.

# Replacement Text for MIMI Content {#content-text}

If the working group takes calling into scope, this section proposes
adding the following to {{I-D.ietf-mimi-content}}, Section 5.10, after
the sentence that puts further discussion of calling out of scope:

> A `session` part names an endpoint; a receiver does not fetch it as it
> would other external content, and the `size`, `expires`, `encAlg` and
> `hashAlg` fields of such a part carry no meaning. Where the media of
> the session named by the part is to be end-to-end encrypted among the
> members of a MIMI room, [this document] describes how the media keys
> are derived from that room's MLS group.

# Implementation Status

This section is to be removed before publication, per {{RFC7942}}.

The argument of {{why}} came out of implementing Section 5.10 in mimid,
an implementation of the MIMI drafts. mimid converts a `session` part to
and from its own front ends and does not fetch it. At the time of
writing mimid implements neither the exporter of
{{I-D.ietf-mls-extensions}}, Section 4.4 nor parent-dependent rooms, so
no part of {{own-room}} has been tested between two implementations, and
this document makes no interoperability claim.

# Acknowledgements
{:numbered="false"}

The call-room note in {{I-D.ietf-mimi-room-policy}}, Section 5 is what
suggested that the shape of the room was the answer rather than an
incidental example.
