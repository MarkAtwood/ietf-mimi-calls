# draft-atwood-mimi-calls

An Internet-Draft saying where a MIMI call's media keys come from.

The pieces of an end-to-end encrypted call are already in the MIMI and
MLS drafts and no document joins them: mimi-content §5.10 carries the
endpoint, mimi-room-policy §5 describes a room for a call, and
mls-extensions §4.4 exports a secret per application component, with
mimi-protocol §10.2 and §10.3 showing the registry such a component
would live in. mimi-content §5.10 then puts further discussion of
calling out of scope, so nothing says which component keys the media.

The argument the draft makes, which the pieces do not make for
themselves: `SafeExportSecret(ComponentID)` takes a ComponentID and
nothing else and is derived from the group's epoch secret, so **every
member of the group can compute it**. Entitlement to a call's media
cannot be established by deriving keys from the group secret; it can
only be established by the membership of the group the secret comes
from. A call among three people in a twenty-person room, keyed from that
room, is readable by all twenty.

So the draft gives a call its own room — the parent-dependent,
fixed-membership shape room-policy §5 already names — and derives the
media secret there. Every epoch change rekeys the call, which means
removing a participant takes their media access with it and needs no
rekeying protocol. For a call among a subset of a room without a room of
its own, it describes distributing a secret with mls-extensions §4.2's
Safe HPKE instead, and what that costs.

Not posted. Nothing here is implemented between two implementations and
no interoperability claim is made.
