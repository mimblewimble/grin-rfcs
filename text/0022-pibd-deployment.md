
- Title: pibd-deployment
- Authors: [Michael Cordner (Yeastplume)](mailto:mc@revcore.net)
- Start date: June 6, 2022
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [mimblewimble/grin#3695]

---

## Table of Contents

- [Table of Contents](#table-of-contents)
- [Summary](#summary)
- [Motivation](#motivation)
- [Reference-level explanation](#reference-level-explanation)
  - [Segmentation, segment creation and responding to PIBD segment request messages](#segmentation-segment-creation-and-responding-to-pibd-segment-request-messages)
    - [Capabilities Flag](#capabilities-flag)
    - [Segment creation](#segment-creation)
    - [Horizon Header Height](#horizon-header-height)
    - [MMR Size Selection and Proof Generation](#mmr-size-selection-and-proof-generation)
    - [MMR Rewinding](#mmr-rewinding)
  - [Requesting Segments and Recreating the UTXO Set](#requesting-segments-and-recreating-the-utxo-set)
    - [Determining the current horizon block](#determining-the-current-horizon-block)
    - [Determining segment heights](#determining-segment-heights)
    - [Peer Selection](#peer-selection)
    - [Requesting output bitmap segments](#requesting-output-bitmap-segments)
    - [Determining starting position for each MMR](#determining-starting-position-for-each-mmr)
    - [Requesting Segments](#requesting-segments)
    - [Receiving and Validating Segments](#receiving-and-validating-segments)
  - [Applying Segments to MMRs](#applying-segments-to-mmrs)
    - [Note about Output/Rangeproof leaf data](#note-about-outputrangeproof-leaf-data)
    - [Appending Hashes and Leaf Data to MMRs](#appending-hashes-and-leaf-data-to-mmrs)
    - [Determining MMR Completion](#determining-mmr-completion)
    - [Post PIBD validation](#post-pibd-validation)
    - [Stopping / Restarting the process](#stopping--restarting-the-process)
    - [Catch Up](#catch-up)
  - [Configuration](#configuration)
  - [Rollout Timing](#rollout-timing)
- [Unresolved questions](#unresolved-questions)
- [Future possibilities](#future-possibilities)
- [References](#references)

## Summary
[summary]: #summary

This RFC describes the rollout of Parallel Independent Block Download (PIBD) functionality on the Grin network. 

The aim of this RFC is to:

1) Provide implementation details and background technical descriptions that are not provided in the original [PIBD Messages RFC](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0020-pibd-messages.md)

2) Formalize the timing of phased releases that enable PIBD on the network and retire the earlier `txhashset.zip` method of fast-syncing new nodes.

## Motivation
[motivation]: #motivation

Previously, the fast-sync process for bringing new nodes onto the network involved downloading and requesting the entire UTXO set up to a 'horizon' block, (i.e. the block below which re-orgs are deemed to be no longer be a concern.) This was transmitted by a single randomly-chosen peer in a single `.zip` file (known as the `txhashset.zip` file). Problems with this approach include (but are not limited to):

* The TXO set is retrieved from a single peer as opposed to many peers on the network
* The download speed of the TXO set is limited to the speed of the randomly chosen peer, leading to wildly different node sync times in practice.
* Downloads must be restarted if interrupted
* Nodes responding to PIBD requests exhibit a noticable pause as a result of needing to mutex lock and zip the entire UTXO set when requested.

PIBD is a new fast-sync method that leverages the unique structure of Grin's backend MMRs to break off and transmit 'segments' of Grin's TXO set independently of each other. This method ensures that

* The TXO set is received from multiple peers in chunks, then validated and pieced together by the syncing node
* The process can be stopped and resumed at will
* The time to process and sync a new node should be more consistent. (Note this doesn't necessarly mean 'faster' in all cases)

Details of the PIBD message protocol, segments and proof generation are defined within the [PIBD Messages RFC](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0020-pibd-messages.md). This RFC will not repeat information found there, but will focus on implementation details that are outside of its scope.

## Reference-level explanation

### Segmentation, segment creation and responding to PIBD segment request messages

#### Capabilities Flag

Nodes capable of responding to PIBD segment request messages advertise the flag `PIBD_HIST_1` denoted by hex `0x40` (binary `0b0100_0000`) in their capabilities field.

Note that there is a previous flag in the core implementation `PIBD_HIST` denoted by hex `0x10` (binary `0b0001_0000`). This flag was rolled out along with segmentation capabilities in Grin 5.0.0 in anticipation of a full PIBD implementation. However, a bug in the original segmentation code unfortunately means this flag is unusable. See https://github.com/mimblewimble/grin/pull/3705 for further details.

#### Segment creation

A node responds to a PIBD segment request by creating a segment of an underlying MMR along with a merkle proof demonstrating inclusion in the complete MMR up to a position represented by the current horizon header. 

Note that the responding node must always respond with segments corresponding to the consensus-agreed horizon header and it is not possible for a node to explicitly request a segment for any other header.

#### Horizon Header Height

The horizon header height is a consensus-derived agreed header below which the UTXO set is deemed to be outside of the possiblity of re-org. (This is the header on which the original `txhashset.zip` method is based.) This can simply be thought of as 'the TXO set as it stood as of a certain block'.

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

Merkle proofs within a segment must prove inclusion in the complete MMR as follows:

* Bitmap segment requests
  * The root of the bitmap MMR at the given height hashed with the root of the output pmmr. 
  * This should give the value of `output_root` in the header.

* Output segment requests
  * The root of the output MMR at the given height hashed with the root of the output pmmr. 
  * This should give the value of `output_root` in the header.

* Rangeproof Segment Requests
  * The root should have the same value as the header field `range_proof_root`

* Kernel Segment Requests 
  * The root should have the same value as the header field `kernel_root`

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
1. [Once each MMR has been recreated to the specified height found in the horizon header, validate all MMR roots and UTXOs](#post-pibd-validation)
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
// Calculate number of leaves in the bitmap mmr
bitmap_mmr_leaf_count = (pmmr::n_leaves(self.archive_header.output_mmr_size) + 1023) / 1024;

// Total size of Bitmap PMMR
bitmap_mmr_size = 1 + pmmr::peaks(pmmr::insertion_to_pmmr_index(self.bitmap_mmr_leaf_count)).last()
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

Upon receiving a requested segment, the node should immediately validate the segment's merkle proof as described in the [PIBD Messages RFC](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0020-pibd-messages.md). Note that validation of kernels and rangeproofs are a separate concern, and can be done after all segments have been downloaded and all MMRs reconstructed.

If the segment fails validation, the node should not immediately ban the peer without checking to see whether the [horizon header](#horizon-header-height) has changed. 

Nodes should cache received and validated segments until their contents are ready to be applied sequentially to their respective MMRs.

### Applying Segments to MMRs

As described in the [PIBD Messages RFC](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0020-pibd-messages.md), segment messages contain lists of hashes, leaves and their positions that should be sequentially applied to their respective MMRs in order to rebuild them.

#### Note about Output/Rangeproof leaf data

Note that leaf data in a segment is always provided alongside a sibling, even if one of the siblings has been pruned (i.e. spent in the case of output/rangeproof MMRs). This means that:

* If both a leaf and its siblings are pruned, a hash (i.e. pruned subtree) will be provided.
* If a leaf is pruned but its sibling has not been pruned, both leaves will be provided, and the node should mark the pruned sibling (as indicted by the output bitmap set) as pruned within local storage.

Although the reasoning behind this is outside the scope of this document, a discussion of why this is the case can be found here https://github.com/mimblewimble/grin/pull/3695#issuecomment-1048770398, (see future possibilities below) TBD

#### Appending Hashes and Leaf Data to MMRs

A high-level description of applying segment data is as follows:

* Sort hash and leaf positions (along with data elements) into their insertion order
* Iterate through all elements. 
	* Skip any hash at a position that has already been applied to the tree (but validate it matches what's been calculated)
	* If the next element is a leaf, append to the MMR as usual
		* If it is a left sibling, append as usual (the next element will be the corresponding right sibling regardless of pruned status)
		* If it is a right sibling, calculate and append hashes up to the next leaf insertion index
	* If the next element to be inserted is a pruned subtree (i.e. a hash value,) add the subtree to the MMR then calculate and append hashes up to the position of the next leaf insertion index. (These hash values that should also be supplied within the segments's list of hashes, so the node should ensure the calculated hashes match the segment's data).

Example:

Assume the current state of the MMR is as follows:

```
Height
1         2    
         / \   
0       0   1 

Size: 3
Last Position: 2
```

The next segment element in the sorted list is a hash (a pruned subtree) at position 5. We append this hash to the MMR, (both leaves underneath it have been pruned)

```
Height
1         2      5  
         / \    / \ 
0       0   1 (3) (4)

Size: 6
Last Position: 5
```

Then calculate hashes up to the position of `next leaf index - 1` (i.e. 7 - 1):

```
Height
2             6
           /     \
1         2       5  
         / \     / \ 
0       0   1  (3) (4)

Size: 7
Last Position: 6
```

The next segment element will be the hash at 6, we check this matches our calculated hash and continue.

At this point, the next elements could be (for example):

* Leaves at 7 and 8 - in which case we'd simply apply the leaves and calculate hashes to the next leaf insertion index, giving:

```
Height
2             6       
           /     \    
1         2       5       9 
         / \     / \     / \ 
0       0   1  (3) (4)  7   8

Size: 10
Last Position: 9
```

* A hash for 9 (if leaves 7 and 8 are pruned), in which case we'd apply the pruned subtree and calculate hashes to the next insertion index, giving:

```
Height
2             6       
           /     \    
1         2       5       9 
         / \     / \     / \ 
0       0   1  (3) (4) (7) (8)

Size: 10
Last Position: 9
```

* A hash at 13 (if leaves 7, 8, 10 and 11 are pruned), in which case we apply the pruned subtree and calculate hashes to the next insertion index, giving:

```
Height
3                     14 
	             /           \
               /               \
2             6                 13   
           /     \          /        \
1         2       5       (9)       (12)
         / \     / \      / \      /    \ 
0       0   1  (3) (4)  (7) (8)  (10)  (11)

Size: 14
Last Position: 13
```

This process continues until all hash and leaf values in the segment have been exhausted.

If there is a mismatch anywhere or an error resulting from applying an element to an MMR, the node should wipe all data and restart the PIBD process. (Note this would most likely be the result of an internal or implementation error)

#### Determining MMR Completion

An MMR can be considered 'complete' if its size matches the corresponding size and MMR root values in the horizon header.

The entire PIBD process can be considered complete if all 3 MMRs are complete, and the horizon header is up-to-date.

#### Post PIBD validation

After validating and applying all segments, the node must verify all transaction objects as per existing fast-sync requirements. This includes.

* Checking all MMR roots
* Validating Kernels
* Validating Rangeproofs

Note that perform performing validation, the node must perform a full pass through the local leaf data of the output and rangeproof MMRs, and ensure any outputs that may have been newly-spent are marked as such (or MMR root validation is likely to fail). This is required particularly when catching-up after a long period of the node being offline, or as a the result of the horizion header changing during the PIBD process.

Note that once a node has validated all TXOs up to the horizon header, it may keep track locally of which objects have already been validated and begin future validations from that point (such as after a long period of being offline).

#### Stopping / Restarting the process

If a node is taken offline during the PIBD process and/or restard, it should resume requesting segments as calculated from the last stored state of each MMR.

#### Catch Up

If a fully-synced node is taken offline for an extended period of time (i.e. longer than the horizon window,) it should attempt to resume PIBD from the last stored state of each MMR. If this is not possible due to the timing of a re-org, it may begin the entire PIBD process from the genesis block.

### Configuration

No explict configuration values need to be exposed for PIBD. However, for the duration of the `txhashset.zip` sunset period, implementations should provide a configuration flag to explicitly disable PIBD and revert back to the `txhashset.zip` fast-sync method.

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

* A "Version 2" of PIBD may be desirable in the future for the following reasons:
   * The transmission of data for pruned siblings is redundant and unnecessary (particularly for Rangeproofs), however this is based on an underlying implementation assumption that must be fully addressed before they can be removed from segment messages. (See https://github.com/mimblewimble/grin/pull/3695#issuecomment-1048770398)
	* The list of output positions for a segment could be considered redundant, as all position data can be derived from the output bitmap.

* The header portion of the overall sync process would greatly benefit from a PIBD-based approach (headers are also stored internally within an MMR)

## References
[references]: #references

Include any references such as links to other documents or reference implementations.

- [PIBD Messages RFC](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0020-pibd-messages.md)
- [Core Implementation Tracking Issue](https://github.com/mimblewimble/grin/pull/3695)
- [PIBD_HIST Segmentation Bug](https://github.com/mimblewimble/grin/pull/3705)