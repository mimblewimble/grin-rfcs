
- Title: pibd-deployment
- Authors: [Michael Cordner (Yeastplume)](mailto:mc@revcore.net)
- Start date: June 6, 2022
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [mimblewimble/grin#3695]

---

## Summary
[summary]: #summary

This RFC describes the rollout of Parallel Independent Block Download (PIBD) functionality on the Grin network. The aim of this RFC is to:

1) Formalize the timing of phased releases that enable PIBD on the network and retire the earlier `txhashset.zip` method of fast-syncing new nodes.
2) Provide implementation details and background technical descriptions that are not provided in the original [PIBD Messages RFC](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0020-pibd-messages.md)

## Motivation
[motivation]: #motivation

Previously, the fast-sync process for bringing new nodes onto the network involved downloading and requesting the entire UTXO set up to a 'horizon' block, below which re-orgs are deemed to be no longer be a concern. This is transmitted by a single randomly-chosen peer in a single `.zip` file (known as the `txhashset.zip` file). Problems with this approach include (but are not limited to):

* The UTXO set is retrieved from a single peer as opposed to many peers on the network
* The download speed of the txhashset is limited to the speed of the randomly chosen peer, leading to wildly different node sync times in practice.
* Downloads must be restarted if interrupted
* Nodes responding to PIBD requests exhibit a noticable pause as a result of needing to mutex lock and zip the entire UTXO set when requested.

PIBD is a new fast-sync method that leverages the unique structure of Grin's backend MMRs to break off and transmit 'segments' of Grin's UTXO set independently of each other. This method ensures that

* The UTXO set is received from multiple peers in chunks, then validated and pieced together by the syncing node
* The process can be stopped and resumed at will
* The time to process and sync a new node should be more consistent. (Note this doesn't necessarly mean 'faster' in all cases)

Details of the PIBD message protocol, segments and proof generation are defined within the [PIBD Messages RFC](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0020-pibd-messages.md). This RFC will not repeat information found there, but will focus on implementation details that are outside of its scope.

## Reference-level explanation

### Segmentation, segment creation and responding to PIBD segment request messages

#### Capabilities Flag

Nodes capable of responding to PIBD segment request messages advertise the flag `PIBD_HIST_1` denoted by hex `0x40` (binary `0b0100_0000`) in their capabilities field.

Note that there is a previous flag in the core implementation `PIBD_HIST` denoted by hex `0x10` (binary `0b0001_0000`). While this flag was rolled out along with segmentation capabilities in grin-core 5.0 in anticipation of a full PIBD implementation, a bug in the original segmentation code unfortunately means this flag is unusable. See https://github.com/mimblewimble/grin/pull/3705 for further details.

#### Segment creation

A node responds to a PIBD segment request by creating a segment of an underlying MMR along with a merkle proof demonstrating inclusion in the complete MMR up to a position represented by the current horizon header. Note that the responding node must always respond with segments corresponding to the consensus-agreed horizon header (and it is not possible for a nodes to explicitly request a segment for any other header).

#### Horizon Header Height

The horizon header height is a consensus-derived agreed header below which the UTXO set is deemed to be outside of the possiblity of re-org. (This is the header on which the original `txhashset.zip` method is based). This can simply be thought of as 'the UTXO set as it stood as of a certain block'.

The current horizon header height is determined by subtracting a state sync threshold (2 'days' worth of grin blocks) from the current header height. This height is then 'rounded down' to a height representing the start of a 12 'grin-block-hour' window, which remains the horizon header for that 12 hour window. The exact calculation of this height is:

```
DAY_HEIGHT: 1440
STATE_SYNC_THRESHOLD = @ 2 * DAY_HEIGHT = 2880
ARCHIVE_INTERVAL = 12*60 = 720

THRESHOLD_HEIGHT = CURRENT_HEADER_HEIGHT - STATE_SYNC_THRESHOLD
HORIZON_HEADER_HEIGHT = THRESHOLD_HEIGHT - (THRESHOLD_HEIGHT % ARCHIVE_INTERVAL)
```

e.g.

```
CURRENT_HEIGHT = 1_000_000
THRESHOLD_HEIGHT = 1_000_000 - 2880 = 997120
HORIZON_HEADER_HEIGHT = 997120 - (977120 % 720) = 99640
```

#### MMR Size Selection and Proof Generation

Nodes responding to PIBD message requests must only respond with with segments and proofs of inclusion for the MMR of the size represented within the current horizon header. These sizes are defined within the header as:

```
output_mmr_size: Output and Rangeproof segment requests

kernel_mmr_size: Kernel segment requests
```

Merkle proofs within a segment must prove inclusion in a MMR as follows:

```
Bitmap segment requests: The root of the bitmap MMR at the given height hashed against the root of the output pmmr. When combined, this should give the value of `output_root` in the header.

Output segment requests: The root of the output MMR at the given height hashed with the root of the output pmmr. When combined, this should give the value of `output_root` in the header.

Rangeproof Segment Requests: The root should correspond directly to the header field `range_proof_root`

Kernel Segment Requests: The root should correspond directly to the header field `kernel_root`
```

Note that it is not possible for a nodes to explicitly request a segment for a particular header; it's assumed that the request is always for the current horizon header.

#### MMR Rewinding

Segments must be created against the UTXO set as it existed at the horizon header, which means that all prunable MMRs (i.e. output and rangeproof) MMRs must be rewound to ensure the leaf inclusion set is as it existed at that block. In practice, implementations only need to rewind the entire txhashset once (on initialisation of the segmenting code or when the horizon block changes), and maintain a representative bitmap set to use in for subsequent PIBD segment requests.

### Requesting Segments and Recreating the UTXO Set

A nodes fast-syncing via the PIBD process will implement the following overall process:

1. [Determine the current horizon block](#determining-the-current-horizon-block)
1. [Determine segment heights](#determining-segment-heights)
1. [Determine peer-selection strategy](#peer-selection)
1. [Request bitmap MMR segments from PIBD-supporting peers](#requesting-output-bitmap-segments)
1. [Determine how much (if any) of each MMR has already been downloaded and applied to the local state](#determining-starting-position-for-each-mmr)
1. [Derive a list of segments needed for each MMR, and request segments in as parallel a manner as possible](#requesting-segments)
1. [Receive and validate each segment](#receiving-and-validating-segments)
1. [Apply each segment to its respective MMR sequentially, essentially re-building the UTXO set](#applying-segments-to-mmrs)
1. Once each MMR has been recreated to the specified height found in the horizon header, validate all MMR roots and UTXOs
1. Move on to block sync, requesting the remaining blocks up to the header height explicitly

Note that this process should work for new nodes (with no MMR data), as well as nodes that were previously synced but have been offline for longer than the horizon window. 

A description of each of these stages as well as relevant implementation details follows.

#### Determining the current horizon block

A syncing node follows the same process outlined [above](#horizon-header-height) to determine the current horizon block. All calculations of required segments will begin with the MMR size values found in this header.

Note that if the horizon header changes (i.e. the 12-hour horizon window 'rolls over') while the node is in the process of syncing, there should be no need to invalidate the partial MMRs that have already been downloaded. The node should:

1. Invalidate all previous segment requests and re-calculate the list of required segments based on the new horizon block
1. Re-request output bitmap segments and recreate a new output bitmap set
1. Continue requesting segments based on the new data and the size of the partial MMRs that have already been downloaded.

Note that if the horizon block changes, outputs in the output bitmap set that were previously unspent could potentially become spent. Before validating MMR roots and UTXOs, the node must compare the entire bitmap set against local MMR storage to ensure outputs have been properly marked as spent and/or pruned locally (see below TBD).

#### Determining segment heights

The [PIBD Messages RFC](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0020-pibd-messages.md) specifies the range of valid segment heights for each MMR, and it is theoretically possible to dynamically adjust segment sizes (and therefore request size) based on network conditions or a host of other factors. However, for simplicity the core implementation currently selects default sizes for each type of segment, and uses them throughout for all calculations. These default values are:

```
BITMAP_SEGMENT_HEIGHT = 9
OUTPUT_SEGMENT_HEIGHT = 11
RANGEPROOF_SEGMENT_HEIGHT = 11
KERNEL_SEGMENT_HEIGHT = 11
```
The processes outlined in later sections will assume constant segment heights, but implementations may choose to adjust these heights and/or perform segment size calculations dynamically.

To determine the number of segments `n` of height `h` required for an MMR of a given size `s`, the following calculation can be used:

```
floor(n_leaves(s) / pow(h, 2))
```

where `n_leaves` is the function that calculates the number of leaves in a pmmr of a given size.

Alternatively, a node can attempt to blindly request segments sequentually until one is returned that is not a 'full' segment, however this strategy may lead to unnecessary requests and difficulties determining how much of the PIBD process remains.

#### Peer Selection

The syncing node should select a different peer for each segment request according to the following strategy:

1. Begin with a list of all known peers
1. Filter this list to only include peers advertising the maximum known difficulty on the network 
1. Further filter this list to only include peers advertising the [PIBD_HIST_1](#capabilities-flag) flag
1. Select a peer at random from the remaining list of peers

For the deployment period between [the initial rollout of PIBD and the retirement of the `txhashset.zip` sync method](#rollout-timing), nodes should abort the PIBD process and fall back to the `txhashset.zip` method of syncing if the following are true:

* There are no peers advertising the maximum known difficulty on the network also advertising the `PIBD_HIST_1` capability
* The node has not seen such a peer for 60 seconds

#### Requesting output bitmap segments

Before any other segments can be validated or applied to local MMR storage, the syncing node must first request and validate the complete output bitmap set corresponding to the output MMR size in the horizon header.

The MMR size, number of leaves and therefore required number of bitmap segments can be pre-determined based on the size of the output mmr in the horizon header, as demonstrated in the following reference code snippet:

```
		/// Calculate number of leaves in the bitmap mmr
		bitmap_mmr_leaf_count =
			(pmmr::n_leaves(self.archive_header.output_mmr_size) + 1023) / 1024;
		// Total size of Bitmap PMMR
		bitmap_mmr_size =
			1 + pmmr::peaks(pmmr::insertion_to_pmmr_index(self.bitmap_mmr_leaf_count))
				.last()
```

The node should request bitmap segments from other peers according to the [Peer Selection Strategy](#peer-selection) above. (The process of re-creating an MMR from segments is described below TBD). Once the bitmap MMR is reconstructed, the node should keep a representation of the underlying bitmap cached in order to facilitate comparison against later incoming segments.

The syncing node is not required to store the contents of the output set bitmap locally (in the core node implementation, the output bitmap set is derived and updated as needed as opposed to being stored locally). If the PIBD process is interrupted, the node may re-request all bitmap segments from peers as part of the normal process of starting/resuming PIBD.

#### Determining starting position for each MMR

Once the bitmap set has been recreated and the node is ready to start requesting segments from peers, the node should examine the contents of its local MMR storage to determine the on-disk sizes of each PMMR. Based on this, it should determine which segments it should request first for each MMR. This ensures that the PIBD process is the same regardless of whether the node is starting from the genesis block, resuming, or catching-up after an extended period of being offline.

#### Requesting Segments

Once the required list of segments has been derived, the node can then begin to request segments in parallel from peers according to the [peer selection strategy](#peer-selection) above. Exactly how the node does this is left as an implementation detail, but the following guidelines should be kept in mind:

* MMRs are an append-only structure, and can only be reconstucted sequentially left-to-right. Therefore there is little point requesting too many segments for a given MMR in advance, particularly if memory is limited.
* A reasonable effort should be made to spread requests across the 3 UTXO MMRs to allow their reconstruction to complete around the same time.

If a requested segment is not received in a pre-determined amount of time, the node may re-request the segment from another peer.

The core implementation uses these values to govern its requesting of segments. Note these are arbitrarily chosen with the aims of keeping memory cache requirements low and avoiding requesting too many segments in advance of what its desegmenter is ready for:

```
/// Maximum number of received segments to cache (across all MMRs) before we stop requesting others
pub const MAX_CACHED_SEGMENTS: usize = 15;

/// How long the state sync should wait after requesting a segment from a peer before
/// deciding the segment isn't going to arrive. The syncer will then re-request the segment
pub const SEGMENT_REQUEST_TIMEOUT_SECS: i64 = 60;

/// Number of simultaneous requests for segments we should make. Note this is currently
/// divisible by 3 to try and evenly spread requests amount the 3 main MMRs (Bitmap segments
/// will always be requested first)
pub const SEGMENT_REQUEST_COUNT: usize = 15;
```

#### Receiving and Validating Segments

* Segment merkle proof validation
* Cache segment locally, ready for insertion into MMRs when ready

### Applying Segments to MMRs

* List of MMR positions and values, these should be applied to the appropriate position at each MMR
* Outputs that are spent should be marked as 'removed' locally.
* If there is a mismatch or other error appling to the MMR, throw it all out (this is likely to be an internal or implementation error)

#### Determining MMR Completion

* Should keep track of representative heights, when all 3 heights match AND the bitmap matches

#### Post PIBD validation

* we check pruning of entire set is correct before validation, leaf set may have changed
* Validate all UTXO objects
* May keep track of state of UTXO object validation up to PMMR height, and mark locally.

### Error handling

* Invalid segments
* Invalid MMR state
* Invalid MMR validation or UTXO

### Configuration

* Existing impls should have a flag in their config files via which PIBD can be turned off, to be removed after the txhashset sunset period

### Rollout Timing

* Alpha released containing full PIBD implementation for testing (testnet only) - Apr 20th, 2022
* Target date for reviewing and merging work performed in `pibd_impl` into master, enabling PIBD on current master for testnet only - July 5th, 2022
* Release of 5.2.0, containing merged PIBD work on testnet only (and a few other features TBD) - Sept 1st 2022
* Release of 5.2.1 - turning on PIBD sync for mainnet as well as testnet - Feb 1st 2023
* Version as yet undetermined: remove support for earlier `txhashset.zip` method of syncing - Feb 1st 2024

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

## Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would be and how it would affect the project and ecosystem as a whole in a holistic way. Try to use this section as a tool to more fully consider all possible interactions with the project and language in your proposal. Also consider how it fits into the road-map of the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities, you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information.

## References
[references]: #references

Include any references such as links to other documents or reference implementations.

- [reference 1](#references)
- [reference 2](#references)
