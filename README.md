# APACK

APACK variant with timestamps.

APACK is  an asynchronous header compression scheme inspired by HPACK.

We assume some familirary with the HPACK compression scheme, but the
text can be read without any detailed knowledge of HPACK.

<!-- vim-markdown-toc GFM -->
* [Background](#background)
* [Overview](#overview)
* [Security Considerations](#security-considerations)
* [Differences from APACK with bitmaps](#differences-from-apack-with-bitmaps)
* [Core Concepts](#core-concepts)
    * [Streams](#streams)
    * [Sessions](#sessions)
    * [Static Encoding](#static-encoding)
    * [Implicit and Explicit Strings](#implicit-and-explicit-strings)
    * [Name-Value List Encoding](#name-value-list-encoding)
* [Data Structures](#data-structures)
    * [The Static Table](#the-static-table)
    * [The Dynamic Table](#the-dynamic-table)
* [Basic Algorithm](#basic-algorithm)
    * [Timestamps](#timestamps)
    * [Encoding with Timestamps](#encoding-with-timestamps)
    * [Updating Decoders Dynamic Table](#updating-decoders-dynamic-table)
    * [Wrapping Timestamps](#wrapping-timestamps)
    * [Resizing the Dynamic Table.](#resizing-the-dynamic-table)
* [Active Reference Sets](#active-reference-sets)
* [Wire Format](#wire-format)
* [Pros and Cons of APACK](#pros-and-cons-of-apack)
    * [Pros](#pros)
    * [Cons](#cons)

<!-- vim-markdown-toc -->

## Background

This is an early draft to gain feedback before possibly propsing the
schema consideration in the IETF QUIC/HTTP standardization for header
compressions.

This text replaces an earlier APACK proposal that used bitmaps posted on the QUIC
mailing list.


## Overview

APACK compresses and transports finite, possibly empty, ordered lists of
name-value pairs. A sesssion is the process of tranporting one list from
the input to an encoder to the output of a decoder.  We are not
concerned with the actual represention of the input or output, only that
it is a list of name-value pairs.

A name or a value is finite possible empty sequence of octets of known
length. We commonly use the term 'string' to refer to such a sequence
and we use the term `field` to present a name-value pair.

A session begins when a name-value pair list is received the encoder and
it ends when the decoder is able to reconstruct the list.

A list could be a HTTP 1.1 header block, but not all lists that APACK
supports could be an HTTP 1.1 header block because HTTP 1.1 does not
support arbitrary octets as names.  Note that an APACK session generally
has a shorter life-span than a HTTP request/response cycle.

The HPACK scheme, unlike APACK, assumes that one session is processed at
time by the encoder and the decoder by transmission over a single
ordered uni-directional stream shared by all sessions.

APACK allows for multiple concurrent sessions concurrent sessions over
separate session specific streams with access to shared bi-directional
control stream.  Note that APACK does not require a session stream to be
bi-directional. APACK could be implemented over a single bi-directional
stream if necessary, but in that case HPACK is preferable.


## Security Considerations

Both APACK and HPACK relies on limited shared state primarily known as
the dynamic table.  To prevent abuse, this table is limited in size. The
encoder dictates current resource consumption and the decoder sets upper
limits out of band. If the encoder does not respect set limits, the
decoder MUST abort.

Compression happens with shared static huffman code points and
references into static and dynamic tables. The static encodings are
generally considered to be safe but the dynamic tables can leak
information across sessions, for exampl by testing if value is already
in the shared state.  Therefore sessions without mutual trust should not
share the same the APACK context.


## Differences from APACK with bitmaps

The original APACK proposal used bitmaps and reference counts, and
assumed that each session (aka exchange) had a uniqueu identifier.  We
know use timestamps instead where a timestamp is a counter, not the wall
clock time. The basic concept remain the same, but the encoding and
lookups are more efficient.

The reason that bitmaps are no longer used is that the same behaviour
could largely be optained simpler with timestamps. Timestamps and
bitmaps both track a range of actually used cached slots in compression
state. The timestamp provides a range and may have more false positives
resulting in fewer slots being available for replacement with new
values. But this can actually be a good thing because the encoder will
then tend to prefer less recently used slots.


## Core Concepts

### Streams

A stream is unidirectional.  There is a session stream (SS) and a control
stream (CS) from encoder to decoder as well as a control return stream (CRS) from
the decoder to the encoder.  We could imagine a session return stream (SRS)
being available, but we do not need it.  We assume that messages are atomic and
ordered with guaranteed delivery on each stream but not across streams.


### Sessions

Recall that a session begins when a name-value pair list is received the
encoder and it ends when the decoder is able to reconstruct the list.

We assume that at session are reasonably short such that it is feasible
to wait for data from one session to arrive before another is processed,
when necessary.  For example, we expect a session to end
before a HTTP request receives the body of a request and before a
response is sent, except when the session is blocked by waiting for
name-value related data on other streams.

There can be multiple concurrent sessions each with a unique SS stream
but there is only on CS-CRS pair.

The encoder processes exactly one session at time and has a global view
of all active sessionss encoder state. 

The decoder can have multiple concurrent sessions but we assume that
messages received on a SS stream are stored temporarily until it is
possible to decode an entire session.  This is not necessarily required,
but it simplifies the discussion.


### Static Encoding

_We assume knownledge of HPACK here._

APACK has a static dictionary and the ability to send name-value pair as
explicity data of the SS stream, neither of which affects any state.
For brewity we skip the details of this encoding but note that the HPACK
approach can largely be reused with allowance for making header fields
compatible with the remaining of the APACK communication frames. We
therefore focus the remaining text on encodings that affect the dynamic
table also known from HPACK. When, occasionally, space or security
considerations make encoding using the dynamic table infeasible or
less preferable, we use the static and explicit communication but
ultimiately allow an implementation to make its own calls.


### Implicit and Explicit Strings

We assume the presence of a static table of name-value pairs and a
static table of huffman codes. We skip the details of this. We also
assume a string may optionally be compressed by a static huffman
encoding scheme but to simplify the discussion we refer to a compressed
or uncompressed strings as an explicit string or an explicit name or
value. An implicit string is a reference to a string stored elsewhere,
and is an important part of both static and dynamic compression. An
implicit string is represented by a reference which is often an index
into some table, but to simplify matters we occasionally use the term
'implicit string' or 'reference'.

It is possible to have an implicit name and an explicit value because we
the decoder might know a name from other values, but not the value we
currently use. We can also have explicit names and explicit values or
implicit names and values. The encoder is free to choose which form it
prefers and whether to use explicit string compression or not, but it
must of course use a form the decoder is able to make use of.


### Name-Value List Encoding

The encoder sends a name-value list as one 'AddField(name, value)'
message per name-value pair on the SS stream in the same order as the
list it aims to encode. The name and value may each be given by as an
implicit or explicit string. 

Then encoder also sends other messages on both the CS and SS stream,
notably a 'Done()' message after the last added field.

The decoder trivially expands the received `AddField` messages once it
is in a state where it is valid to do look up the implicit references
that may be present and decompresses any compressed explicit strings
that may be present.

The wireformat of each command is discussed subsequently.

The main job of APACK is to ensure that the proper implicit references
are valid at the right time.  This is done by updating a data structure
known as the dynamic table (DT) at both the encoder and at the decoder
in a way that both parties can agree upon even if their respective views
are not in perfect sync. An implicit string, or value, is thus a means
to locate a value in the dynamic table, or alternatively, in a
pre-populated static table (ST). When we thus say implicit string, we
assume that it is a reference which is encoded such that it can
correctly select an entry in either DT or ST.

In comparison, HPACK has an easier job because there is only one ordered
stream, so the decoder always has the same view as the encoder, albeit
after a delay, but other than that HPACK does the same job as APACK by
maintaining the dynamic table in a timely fashion.

The static table is pre-populated out of band by name-value pairs that
makes sense to the specific application. Likewise, the static huffman
code points for explicit string compresion is pre-populated according to
the specific application.


## Data Structures

### The Static Table

The static table (ST) is similar to the static table usd in HPACK. It is
a table of name-value pairs where both content and size is fixed and
upgreed upon out of band between the encoder and the decoder. Each entry
is given an index between 1 and K where K is the table size. 

A reference to a name or a value is given by the index into the table
and whether it is a name reference or a value reference, or both.

Note that names and values are not stored separately because we largely
expect values to be specific to one name.

There can be multiple entries with the same name. They can even have the
same value, although, for static table, that would be pointless.


### The Dynamic Table

The dynamic table (DT) known from HPACK is also found in APACK in a
modified form.

DT operates as a fixed size that both parties agree upon.  This means
that any new entry must replace an old entry wether it contains valid
data or not.  We say that the old value is evicted when a new value is
updated. The DT is limited in size depending on use case, but for HTTP
headers it could be less than 1000 entries. The exact representation is
not important to our discussion, only the operations that we can
perform on it, and the data that we can store in it.

Each location in the DT is called a slot.  A slot contains an explicit
name and an explicit value.  The content of a slot is called an entry,
so when a slot is updated the old entry is evicted and replaced by a new
entry.  Given a slot it is not naively possible to identify
which entry is meant, and this is the main problem for APACK to solve.

Both the encoder and the decoder has a copy of the DT, but obviously the
decoder has a delayed view.  Updates to the decoders copy of DT happen
via messages over the control stream CS. Therefore the decoders copy is
a delayed version of the encoders copy.

So far this is the same representation as HPACK but each slot also has
an additional APACK specific record that will be discussed subsequently.

HPACK uses FIFO re-indexing after each update such that the most recent
update have the lowest possible index and compresses integers such that
the latest update is the most compact reference. We cannot assume that
such a reference can be used with APACK directly because the aynchronous
sessions would not necessarily agree upon when re-indexing takes place.


## Basic Algorithm

The encoder only processes one session at a time and does not wait for a
response from the decoder before starting the next queued up session, if
any.  Each session started is conceptually added to the set of active
sessions.  Periodically the decoder reports back which sessions are no
longer active. 

Each active session has a number of implicit string references into the
dynamic table DT. For the sake of discussion we assume that no
references are into the static table unless explicitly stated, thus by
number of active references we mean number of dynamic references from
active sessions.

An active session is a session that the encoder believes might not have
been completed by the decoder.  The set of active sessions is a
consertative estimate that is updated when a new sessions is processed
by the encoder, and and when a relevant message is received from the
decoder on the CRT stream.

The encoder conceptually keeps track of the set of active references and
retires references that originate from sessions that are known to no
longer active at the decoder. Since the encoder only processes on
session at a time active sessions mean those sessions the encoder
believes might still be active at the decoder. The set of active
references is not maintained in explicit form, as we shall see later on.
Neither is the set of active sessions.

When the encoder decides to update a slot and thereby evict an old entry
it first makes sure that the old entry is not referenced by any active
references.  This makes it perfectly safe to modify, but it does not in
itself make it safe to reference because the update happens on the CS
stream and a reference happens asynchrounously on a SS stream so the
decoder risk seeing the old value if not being careful even if there is
only one active session.

We manage the set of active references via timestamps and timestamps
also solve the problem of managing the set of active references
efficiently and contribute to solving the problem of FIFO re-indexing
references such that we can use low numbers to reference recent indexes.

Once we are able to resolve reference ambiguities and to efficiently
maintain the set of active references, the APACK algorithm is complete
and only details of wireformat remains.


### Timestamps

Each session presumably has an application specific session ID
associated with each SS stream, but we prefer to use our own internal
identification which we refer to as a timestamp although it really is a
counter.

We do not assign a unique timestamp to all sessions. The encoder
maintains an update timestamp TSU which is incremented before each
session that make updates to the dynamic table. Other sessions do not
affect TSU but all sessions are given a session timestamp TSS which is
the current TSU value when the session is encoded.

Each slot in the dynamic table has an APACK record which holds a TSU and
a TSR timestamp. slot.apack.TSU is the timestamp at the time the slot was last
updated. slot.apack.TSR is timestamp when the slot was last referenced.

When the encoder updates a slot in DT is sends an 'UpdateField(name,
value)' message on the control stream CS. After the last update message
has been sent for a given session, a 'SyncUpdate(TSU)' is sent on CS. If TSU
is not modifed, no 'SyncUpdate(TSU)' is sent.

When an encoder session references a slot in the dynamic table it sets
slot.apack.TSR = TSS. If it also updates the slot it must do so before
referencing it and then slot.name, slot.value and slot.apack.TSU must be
updated. Therefore the TSU and TSR values are normally equal initially.

It is valid for the encoder to update a slot without immediately
referencing it, for example to populate the DT with anticipiated future
sessions. In this case it may, but need not, update the TSR. Updating
the TSR reduces the chance of early eviction.


### Encoding with Timestamps

When the encoder starts sending a sequence of 'AddField(name, value)'
messages on the current SS stream it first sends a 'SyncRef(TSS, TSU)'
message with the current TSS timestamp and the largest TSU value seen
among all referenced DT slots.  If there are no references into the
dynamic table, the sync message may be skipped.

_(If it is not convenient to pre-scan before the encoder starts to 
send we can also send the a TSR value with each 'AddField' message
which allows for earlier decoder processing at the cost of more space.
The encoder could also defer sending the 'SyncRef' message until the
end if the decoder agrees and if the decoder caches all previous
messages on SS. In the following we assume that SyncRef is sent first if
present.)_


Invariant (I): The decoder MUST NOT attempt to look up any 'AddField'
reference in the DT before it has seen a 'SyncUpdate' value with a TSU
value larger than or equal to the TSu value received on the SS stream.

The simplest way to uphold the invariant (I) is to see if the decoder has
received such a TSU value and otherwise enqueue the session ordered by
timestamp.  We assume this approach is taken in the following, although
implementations may vary as long as the invariant is uphold. If no
'SyncRef' was received, the decoder can immediately process the session
without using the dynamic table and it can also safely defer processing
until any later point in time.

When the decoder processes a session we distinguish between starting to
read the SS stream and starting to dereference added fields.
Dereferencing is best done ASAP for example by copying referenced fields
or by fully expanding the list for the end consumer. It is safe to
arbitrarily defer dereferencing and/or to do so concurrenlty with other
sessions, but the sooner the decoder completes dereferencing, the sooner
the encoder will be able to reuse slots.

When the decoder has finished dereferencing all fields of a session, it
sends a 'Commit(TSS)' message on the CRS return control stream. 

Invariant (II): The encoder MUST NOT attempt to reuse a slot for which
there are still uncomitted TSS timestamps.

Invariant (II) may be upheld as follows: When the encoder sends
'SyncRef(TSS, TSU)' it also increments a counter for that TSS value in
an internal map called TSS-Map. When the decoder receives a 'CommitRef'
message on the CRS stream it decrements the entry in the TSS-map.  A
TSS-Map entry that decrements to zero may later increment depending on
new encoding sessions. Instead of using a counter, a multimap could also
be used such that an arbitrary matching key is removed upon receipt.

The encoder MUST protect itself from malicious access patterns into , for
example by limiting the number of entries if it uses a naive list as
map and abort if an entry is not found.

The decoder MUST protect itself from malicous access patterns on the
queue waiting for the proper TSU value for 'UpdateSync'. Normally a
simple ordered list will do due to the natural access pattern, buf if
very long repeated scans are identified, this might be an indicator of
abuse.

The encoder maintains the minimum TSS value that is not zero in the
TSS-map. When the encoder needs to choose a new slot to update, it scans
the DT table for a slot with a slot.apack.TSR value below this minimum.

The allocation scan can start from the slot after the last updated
slot which results in eviction of the least recently updated slot first.
The encoder can also use other heuristics such as indexing slots by TSR
and update the slot with the minimum TSR value which results in evicting
the slots that are rarely used while avoiding slots that might have been
update a long time ago, but which are still in active use.


### Updating Decoders Dynamic Table

So far we have only discussed when it is possible to update the encoders
table and when it is safe to access the decoders table DT. We still need to
specify how the decoder actually handles updates to DT.

We can largely split the decoder into two parts. The one that handles
traffic on SS streams and commit messages on the CRS stream, and one
that handles inbound traffic on the control stream CS. We will consider
this latter part now.

When a 'SyncUpdate' message is received the decoder registers this value
as current and goes on to look for queued up sessions that wait for a
sync message at or below this value. If any are present, the decode may
choose to process these sesions now, because it is valid to do so, or it
may move them to some other queue that can be processed when convenient.

When a 'UpdateField' mesage is received, the DT is immediately updated
with the new contents. ; The decoder does not need any APACK specific
record in DT slots, unlike the encoder. It is always valid to modify an
entry in the DT because the encoder only use slots that are not yet
referenced.

If an 'UpdateField' message references an invalid slot, the decoder must
abort immediately abort.


### Wrapping Timestamps

It is fairly easy to handle timestamp wrapping if we are sure that all
currently relevant timespans are within a range less than half the
representation size, such as 127 units when stored in a byte. When this
is the case we can consider the given timestamp as a module to a larger
internal represention that is too large to wrap.

The trick is to ensure that we do not have any too old timestamps.

We can limit the number of concurrent sessions to a hard number such as
8000 and use sessions without 'SyncRef' messages when reaching this
limit, or simply holding back such sessins at the implementers
discretion. With a limit of, say, 30000 we can safely use a 16-bit
timestamp and we encode it compressed on the wire.

The encoder always grows timestamps incrementally for each new session.
The receiver might receive sessions out of order, but they will always
be among the set limit of active sessions because the encoder will wait
for commit messages before starting more sessions that make use
'SyncRef' messages. Other sessions are stateless and need not be
concerned with timestamps at all.

Thus we can limit the extra space usage on each slot to 32 bits, 16 bits
for the TSU timestamp and 16 bits for the TSR timestamp.

We can limit the range if our dynamic table size is comparatively small.
The timestamp only increments with each update, and any TSR refernece
timestamp cannot be below any valid TSU timestamp because a slot cannot
be referenced before it is updated. Therefore we only need just over
double the dynamic table limit for the timestamp representation size.

If we set the upper limit to 30000 entries, we can still have a 16-bit
timestamp without having to limit the number of concurrent sessions.

If we use an 8-bit timestamp we surely need to limit the number of
concurrent sessions, perhaps more so than desirable, but it might be
useful on resource constrained systems.


### Resizing the Dynamic Table.

The encoder can send a 'Resize(N)' message on the control channel to
indicate that it does have any active references at slots above the
given limit N and does intend to until otherwise advised.

If the encoder sends an invalid resize message, the decoder would
eventually try to resolve a reference outside the given limit. Such an
action must be detected and MUST lead to abort.

If the given limit is above the out of band agreed limit set by the
decoder, the decoder MUST abort.


## Active Reference Sets

This section is purely informative to help understanding the underlying
concepts and to motivate the use of timestamps for set management.

The APACK timestamp algorithm works by clustering active references into
buckets named by timestamp. The TSU timestamp is an upper bound on the
range of active timestamps and the TSS timestamp is a lower bound.

By only evicting entries from slots with references below the smallest
TSS among active sets we are sure to not use any slot that might be
actively referenced.

The active sets are effectively grouped into buckets of TSS timestamps
such that if many sessions use the same dynamic values but do not
introduce any new values to cache in the dynamic table, then they will
all use the same TSS timestamp and can be tracked with a simple counter. 

We risk having many false active references if old sessions are not
timely consumed by the decoder but it is not really problematic,
possibly even beneficial, until a point. This is because those false
positives are slots that are likely to be used again soon, compared to
those slots that are currently below the TSS limit. Only when the
remaining slots are few in numbers will we start to see a massive cache
churn, and then the decoder should speed up, or operate with a larger
cache.

We could also use bitmaps to track the sets of active references per
session, but we do not really need that granularity and hence timestamps
provides a simpler approach to solving the problem.


## Wire Format

TODO

Not fully decided, but use something similar and preferable compatible
with HPACK.

The main issue is how to efficiently encode references into DT since we
cannot trivially use the FIFO principle.


## Pros and Cons of APACK

### Pros

APACK uses two timestamp fields for each entry in the dynamic table
and a map of currently active sessions that have not yet commit.

There is a minimum of coordination between encoder and decoder. The
decoder only needs to eventually send a commit message for each session,
but it is critical when it does so, to a limit. There is great freedom
in choosing alternative heuristics because the timestamps provide useful
information including the ability to populate the dynamic table ahead of
time if so desired.

The message structure is very similar to HPACK except for a few extra
sync messages, the fact that communication happens over multiple
streams and that FIFO indexing is being used.

The fact that the operations are asynchrounous does not impact the
encoder much while the decoder has a simple choice to make between
immediately processing a session or placing the session on a sorted list
for later processing when an appropriate sync message arrives on the
control stream. Once the decoder is cleared to process a session, it can
do so at its own leasure, within reason, because the data structures it
depends upon will remain valid.

In its simplest form APACK only requires two extra field and a linked
list to handle commits on the decoder side and a linked list to handle
sync coordination decoder side. Such a simple implementation requires
safeguards against abuse, but will otherwise work fine for typical
access patterns.


### Cons

The integer encoding of references is not the smallest possible because
HPACKs FIFO indexing is not practical, but something similar could be
obtained at the cost of more complexity.

There is an element of blocking, especially if some sessions sends a lot
of updates on the control stream, such as many large cookies that keep
changing. If this is a signficant problem, it would be possible to
allocate a second control stream to low priority large values but again,
this increases complexity and might not work if all sessions use large
values.
