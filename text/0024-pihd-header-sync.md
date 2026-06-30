- Title: pihd-header-sync
- Authors: wiesche89
- Start date: June 29, 2026
---

## Summary
[summary]: #summary

Parallel Initial Header Download (PIHD) lets a node request deterministic block
header segments from one or more peers during header sync, alongside legacy
header sync.

This RFC adds two p2p messages: a request for a header segment and a response
with the headers. Peers signal support through a new capability bit. Nodes that
do not support PIHD continue to use legacy header sync.

## Motivation
[motivation]: #motivation

Header sync is the first step of node synchronization. The current
locator-based approach is simple, but it is sequential and usually talks to one
peer at a time. This can make initial sync slower when a node is far behind the
network tip.

PIBD already requests independent segments for later sync stages. PIHD applies
the same idea to block headers. Headers form a linear chain and are already
transferred in bounded batches, making deterministic segmentation
straightforward.

PIHD does not replace legacy header sync. It adds a parallel path when
PIHD peers are available. Legacy header sync stays available as a fallback.

## Community-level explanation
[community-level-explanation]: #community-level-explanation

A syncing node needs all historical block headers before it can move to the
next sync stage. Traditionally it asks a peer for headers after a locator and
continues from there. With PIHD the node splits the missing header history into
fixed-size segments. It can ask different peers for different segments.

Each segment is identified by a segment height and an index. For PIHD the
segment height is fixed, so each full segment contains 512 headers, matching
the maximum header batch size.

When peers with PIHD support are available, the node requests header segments in
parallel. Segments can arrive out of order. The node temporarily buffers them
until the next expected segment is ready. If PIHD stalls or peers do
not provide valid data, the node falls back to legacy header sync.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A header segment is identified by the tuple `(h, i)`, where `h` is the segment
height and `i` is the zero-based segment index. Here, `h` is the PMMR segment
height from `SegmentIdentifier`, reused from PIBD. It is the binary logarithm
of the segment capacity, not a block height.

For PIHD, `h` is fixed to `9`. The segment capacity is therefore:

```text
2^9 = 512 headers
```

This value matches `MAX_BLOCK_HEADERS`. Larger header segments do not fit the
current header message limits, and smaller segments add round trips and sync
state without improving throughput. Peers should reject PIHD header segment
requests with any other segment height.

The first block height for segment `(h, i)` is:

```text
i * 2^h + 1
```

The last block height is the first height plus the segment capacity minus one,
capped at the serving peer's header head. Segment index `0` therefore starts at
height `1`. The genesis header is not requested as part of PIHD. The final
segment may contain fewer than `MAX_BLOCK_HEADERS` headers when it reaches the
serving peer's header head. A request starting above the serving peer's header
head returns an empty segment.

Peers signal support for serving deterministic header segments with a new
capability bit:

```text
PIHD_HIST
```

In the reference implementation this capability is part of the default
capability set, so nodes advertise PIHD support by default.

Nodes use this capability to avoid sending the new message types to peers that
do not advertise support. Capability gating avoids a protocol version bump for
peers that do not use these messages.

### Header segment messages

This RFC introduces two p2p messages.

The request contains a `SegmentIdentifier`, using the same encoding as PIBD:

  * segment height `h` (1 byte)
  * zero-based segment index `i` (8 bytes)

The response contains:

  * segment height `h` (1 byte)
  * zero-based segment index `i` (8 bytes)
  * number of headers `N` (2 bytes)
  * list of `N` block headers

The response header count must not exceed `MAX_BLOCK_HEADERS`.

The new p2p messages are:

ID | Name | Message type
--- | --- | ---
29 | GetHeaderSegment | Request
30 | HeaderSegment | Response

### Serving header segments

A node should only serve header segments to peers that advertised the PIHD
capability. On a request, the serving node checks that the requested segment
height matches the fixed PIHD height. It then returns the deterministic header
range for the requested index. Here, deterministic is relative to the serving
peer's current header chain. A peer on another fork may return different
headers for the same segment identifier, because the identifier commits only to
a height range and not to a chain anchor.

Malformed or unsupported requests should be ignored. Serving nodes also limit
header segment requests per peer to avoid request spam.

### Requesting header segments

This section describes the reference requester. The protocol does not mandate a
particular scheduling strategy.

A syncing node selects peers that advertise PIHD support, have more total
difficulty than its current header head, and are near the network tip. It then
sends several header segment requests in parallel. The node tracks which peer
each segment was requested from.

Each response is checked against the requested segment identifier and the
expected header window. The headers must start at the expected height and must
be contiguous. Valid segments are applied to the header chain in index order.
Segments that arrive ahead of the next expected index may be buffered briefly and
applied later.

Unsolicited or duplicate responses are dropped. Peers that return segments
violating the PIHD segment rules may be banned.

If header segment requests repeatedly time out and PIHD makes no header progress
for a fixed period, the node disables PIHD temporarily and falls back to legacy
header sync. PIHD is retried automatically once the disable period elapses.

### Security and validation model

PIHD header segments cannot be verified independently, unlike PIBD segments.
The parallelism is in fetching header ranges. Validation still happens in chain
order. A node applies ready segments only when they connect to the current
header head. The headers go through the same proof of work, difficulty, and
chain validation rules used by legacy header sync.

Peers that send segments with the wrong segment height, wrong start height,
headers that are not contiguous, or headers that do not validate may be
rejected or banned. The security assumption is the same as sequential header
sync: headers are accepted only by validating them against the local chain
state.

### Sync status

The reference implementation records whether header sync is using PIHD or
legacy header sync and exposes this through the node status interfaces.

## Drawbacks
[drawbacks]: #drawbacks

PIHD adds another header sync path and some additional sync state. A node must
track pending segment requests, handle out-of-order responses, and apply
segments only when they connect to the current header head.

The fixed segment height keeps the protocol small, but it also means the segment
layout is part of the PIHD protocol. Changing it later would require a protocol
or capability update so that old and new peers do not interpret the same segment
index differently.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The main alternative is to keep only legacy header sync. This keeps the
implementation smaller, but it does not allow the node to make progress from
several peers in parallel during the header sync stage.

Another alternative is to allow variable header segment heights. This was not
chosen. The fixed height maps exactly to the `MAX_BLOCK_HEADERS` limit.
Smaller values provide no clear benefit and larger values require a larger
wire-format change.

Legacy header sync remains available and is used whenever PIHD is not
available or does not make progress.

## Prior art
[prior-art]: #prior-art

PIHD follows the segmentation model introduced by PIBD in RFC 0020, but applies
it to block headers.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

The sync parameters, such as in-flight limits, timeouts, and fallback windows,
are currently fixed at compile time. It remains open whether any of these should
be exposed as configuration parameters.

PIHD is advertised by default in the reference implementation. This may be
revisited if deployment shows that an opt-in rollout is preferable.

Peer selection and fallback behavior may need more tuning on small networks,
or on networks where only a few peers support PIHD.

## Future possibilities
[future-possibilities]: #future-possibilities

Future versions may adjust the header segment size or negotiate different
segment layouts, but such changes should be treated as protocol changes.

Header segments that can be verified on their own, based on a committed header
MMR or checkpoint, would be a stronger design. This would require a separate
consensus-level change.

Request scheduling, peer selection, cache limits, and fallback behavior can be
adjusted after more network use.

## References
[references]: #references

  * [1] RFC 0020: PIBD messages
  * [2] Reference implementation: [PIHD header segment protocol](https://github.com/mimblewimble/grin/pull/3879)
  * [3] Reference implementation: [PIHD header sync](https://github.com/mimblewimble/grin/pull/3882)
