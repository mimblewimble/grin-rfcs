
- Title: pibd-messages
- Authors: [Jasper van der Maarel](mailto:j@sper.dev), [Antioch Peverell](mailto:apeverell@protonmail.com)
- Start date: September 14, 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

This RFC defines new peer-to-peer messages that are required to enable a novel sync method.
This method reduces node bootstrapping time and increases overall robustness of the sync process.
It introduces the concept of "segments", which are packets of self-contained partial state that can be downloaded and verified independently from each other.
After downloading all segments and performing some additional verification steps, the full state can be reconstructed. 

# Motivation
[motivation]: #motivation

In Grin, the state of the chain at a certain block is defined by the set of all historical kernels, the unspent outputs and the corresponding rangeproofs.
This state is stored in four distinct (Prunable) Merkle Mountain Ranges ((P)MMRs): the output bitmap MMR, the output PMMR, the rangeproof PMMR and the kernel MMR.
In the old sync process, all of this data was packaged in a single zip file and downloaded from a single peer.
The incoming data could only be verified after the full zip file had been downloaded.
Because of the reliance on a single peer at a time, it is bottlenecked by the upload bandwidth of that peer.
It is also a single point of failure.
Any corrupted data or connection interruption potentially wastes a lot of time, as the download has to be started from scratch from another peer.

Using the eight new peer-to-peer messages introduced in this RFC, we no longer have to rely on the download of the txhashset.zip file.
Instead, bootstrapping nodes can download segments of the MMRs simultaneously from multiple peers and use them to reconstruct the full state.
Malicious peers are detected quickly by virtue of each segment being independently verifiable.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

As previously mentioned, the chainstate at a certain block is represented by the four MMRs.
The PMMR [\[1\]](#references) is a datastructure similar to a Merkle tree, which allows us to efficiently prove membership of an element inside the tree.

It allows us to calculate a single hash representing the entire dataset: the root.
The block header stores the roots of the different MMRs, committing to the state at that point in time.

The peer-to-peer messages introduced in this RFC enable us to split the MMRs up into segments.
Each of those segments contains a subset of the MMR data, accompanied with a membership proof in the MMR.
The proof allows us to recalculate the MMR root using the segment data.
If the MMR root matches the hash committed to in the block header, the segment is valid.

Bootstrapping nodes will download all segments of the four MMRs in parallel and reconstruct the full MMR from them.
After some additional verification steps the new state is accepted and the node is synced.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This RFC introduces the concept of MMR segments, which are subtrees of the full MMR, along with hashes of pruned subtrees and a Merkle proof.
For each of the four relevant (P)MMRs (output bitmap, output, rangeproof and kernel) we create 2 new peer-to-peer messages: a request and a response.

Clients can signify support for segment requests by signaling it in their capabilities using a new bit flag: `PIBD_HIST = 1 << 4`.

The reference implementation for this RFC can be found in [\[3-6\]](#references).

## MMR segments
[mmr-segments]: #mmr-segments
We define a (P)MMR **segment** at segment height `h` as a set of `N = 2^h` consecutive leaves, along with the necessary data to verify membership of the leaves in the original MMR.
Each segment is consecutive with the previous and is identified by the tuple `(H, h, i)` where `H` is the block hash, `h` is the segment "height" and `i` is a zero-based segment index: segment `(H, h, i)` contains the non-pruned leaves with leaf position in the interval `[i*N, (i+1)*N)` of the MMR state at the block with hash `H`.
The final segment is allowed to have less than `N` elements because the full MMR does not necessarily contain a multiple of `N` leaves.

Concretely, a segment contains the following items:
- The identifier tuple `(H, h, i)`.
- List of MMR positions and corresponding list of intermediary hashes, sorted by MMR position in ascending order.
At a minimum it should contain all intermediary hashes that are required in order to reconstruct the subtree root.
These lists are ignored for non-prunable MMRs.
- A list MMR positions and a corresponding list of leaves in the segment (leaf index range `[i * 2^h, (i+1) * 2^h)`), sorted by MMR position in ascending order.
At a minimum it should contain all unpruned leaves and the direct sibling of every unpruned leaf.
- [The segment Merkle proof](#segment-merkle-proof).

After receiving a segment, clients should verify if its contents are contained in the MMR by performing the following two steps: 
1) calculate the [root of the segment](#segment-root) (using the leaf data and intermediary hashes) and 
2) reconstruct the full MMR root using the segment root and the [segment merkle proof](#segment-merkle-proof) and compare it to the root in the block header.

Clients are allowed to request fully pruned segments.
In case of a fully pruned segment the list of leaves is expected to be empty and the list of hashes to contain a single entry: the first unpruned parent of the segment root.
The proof is expected to contain the required hashes to reconstruct the MMR root from the first unpruned parent onwards.

Clients should only ever request or respond with segments for the latest and first-to-latest canonical block for which `height % TXHASHSET_ARCHIVE_INTERVAL== 0`.

Clients are free to choose a segment size by setting the height within a certain range in their requests.
This allowed range is set differently depending on the MMR contents.
The following table has concrete values for the different MMR types and the allowed height range.
The upper bound has been chosen such that a single segment roughly corresponds to the maximum size of a block (1,348,032 B).
The lower bound is set such that the segment size is roughly an sixteenth of the maximum size.

| MMR Type      | Prunable? | Leaf size (B)   | Height range | Maximum segment size (B) |
|---------------|-----------|-----------------|--------------|--------------------------|
| Output bitmap | No        | 128             | \[9,13]      | 1,114,177 + proof        |
| Outputs       | Yes       | 34              | \[11,15]     | 1,376,353 + proof        |
| Range proofs  | Yes       | 675             | \[7,11]      | 1,398,881 + proof        |
| Kernels       | No        | ~114 (variable) | \[9,13]      |   999,489 + proof        |

Because the block header only commits to `H(output_root|bitmap_root)` [\[2\]](#references) as opposed to each of the roots separately, the output bitmap segments and output segments require an additional field to specify the root of the other MMR. 

### Segment root
[segment-root]: #segment-root
A segment can be viewed as a valid MMR with its starting position shifted.
Therefore we define the segment root analogous to an MMR root [\[1\]](#references): we iteratively hash the peaks in the segment from right to left.
However instead of including the segment size in the hashing calculation, we substitute it with the full MMR size.
This ensures that the root of the final segment is the same as the bagged peaks on the full MMR, greatly simplifying our merkle proof.

The segment root can be calculated by iterating over all MMR positions in the segment range.
If the position is a leaf: check (for prunable MMRs) to see if the element is expected to be there (based on presence of the element or its sibling in the bitmap) and if it is get it from the list of leaves, hash it and store the hash in a list of temporary hashes.
For all other positions: if both children are not present in the list of temporary hashes (i.e. they are both pruned), do nothing.
If either or both hashes are present in the temporary list, hash them together and store the new hash in the temporary list.
If only one of the children was present, obtain the hash of the other child from the list of hashes in the segment instead.
After looping through all positions, we are either left with 1 entry (full segment) or multiple entries (partially filled, final segment) in the list of temporary hashes.
If there are multiple, bag them together.
This leaves a single hash, the segment root.

### Segment merkle proof
[segment-merkle-proof]: #segment-merkle-proof
Whenever a node receives an MMR segment, it needs to be able to verify that its contents are in fact part of the MMR.
This is usually done by providing a merkle proof, which in short is a list of hashes that, combined with a leaf hash, is used to calculate the root.
If it successfully reproduces the root, the proof is considered to be valid.

It would be very inefficient and redundant to provide a separate merkle proof for each leaf in the segment.
Therefore we create a single optimized merkle proof per segment, based on two observations: firstly, a full segment (`2^h` elements) forms a full binary subtree in the MMR. 
This means it has a single root. 
Secondly, the final segment is potentially only partially filled (number of elements is not a power of 2). 
If this is the case, each of the peaks in the segment are direct peaks in the MMR.

As such, a segment merkle proof is simply a list of hashes:
1. The siblings along the path from the subtree root (final MMR position of the segment) up to its corresponding peak in the MMR
2. Peaks to the right of our subtree, bagged together to a single hash
3. Peaks to the left (from right to left) with a position smaller than the first MMR position of the segment 
For efficiency reasons the peaks on the right side of our subtree are bagged together.
Note that this procedure will also behave as expected for the partially filled final segment, since in that case step and 1 and 2 will not produce any hashes and step 3 will give us all the other peaks in the MMR.

Verifying a proof is done by iteratively hashing our segment root together with the hashes in the list above.
If the final hash corresponds to the MMR root, the segment is considered valid. 

### p2p messages
[p2p-messages]: #p2p-messages

This RFC introduces 8 new peer-to-peer messages that follow request-response semantics: 2 for each MMR type.

Each request contains the segment identifier triplet `(H, h, i)`:
- block hash `H` (32 bytes)
- segment height `h` (1 byte)
- zero-based segment index `i` (8 bytes)

The corresponding reference impl Rust type is `SegmentRequest` [\[5\]](#references).

The response is a segment as explained in the [previous section](#mmr-segments):
- block hash `H` (32 bytes)
- segment height `h` (1 byte)
- zero-based segment index `i` (8 bytes)
- length of intermediary hashes list `N_h` (8 bytes)
- list of intermediary hash positions (`8 * N_h` bytes)
- list of intermediary hashes (`32 * N_h` bytes)
- length of leaf list `N_l` (8 bytes)
- list of leaf positions (`8 * N_l` bytes)
- list of leaf data (variable, depends on leaf data type)
- segment merkle proof: number of hashes `N_m` (8 bytes) and the list of hashes (`32 * N_m` bytes)
- _OutputSegment only:_  root hash of the output bitmap MMR at block hash `H` (32 bytes)

The corresponding reference impl Rust types are `SegmentResponse<T>` and `OutputSegmentResponse` [\[5\]](#references).

The output bitmap segments use a specialized (de)serialization scheme instead.
First, consider all the leaves in the segment as belonging to a single continuous bitmap.
Split this bitmap up in blocks of `2^16` bits each.
Each block is (de)serialized independently, with different encodings depending on the block cardinality (reference impl. [\[6\]](#references)).

First, start of by writing the following data:
- number of chunks contained in the block `N_ch <= 64` (1 byte)

If the cardinality is less than 4096, write the following data:
- serialization mode `1u8` (1 byte)
- cardinality `N_pos` (2 bytes)
- list of positive positions (`2 * N_pos` bytes)

If `2^16 - cardinality = N_neg` is less than 4096, write the following data instead:
- serialization mode `2u8` (1 byte)
- "inverse" cardinality `N_neg` (2 bytes)
- list of negative positions (`2 * N_neg` bytes)

In any other case, write the following data instead:
- serialization mode `0u8` (1 byte)
- full bitmap as raw bytes (`128 * N_ch` bytes)

The full output bitmap segment response is defined as followed:
- block hash `H` (32 bytes)
- segment height `h` (1 byte)
- zero-based segment index `i` (8 bytes)
- number of blocks (2 bytes)
- list of blocks (variable \# of bytes)
- segment merkle proof: number of hashes `N_m` (8 bytes) and the list of hashes (`32 * N_m` bytes)
- root hash of the output PMMR at block hash `H` (32 bytes)

The corresponding reference impl Rust type is `OutputBitmapSegmentResponse` [\[5\]](#references).

Finally, the new p2p message are as follows:

| ID | Name                   | MMR type      | Message type | Rust ref. impl. type |
|----|------------------------|---------------|--------------|-----------|
| 21 | GetOutputBitmapSegment | Output bitmap | Request      | `SegmentRequest` |
| 22 | OutputBitmapSegment    | Output bitmap | Response     | `OutputBitmapSegmentResponse` |
| 23 | GetOutputSegment       | Output        | Request      | `SegmentRequest` |
| 24 | OutputSegment          | Output        | Response     | `OutputSegmentResponse` |
| 25 | GetRangeProofSegment   | Range proof   | Request      | `SegmentRequest` |
| 26 | RangeProofSegment      | Range proof   | Response     | `SegmentResponse<RangeProof>` |
| 27 | GetKernelSegment       | Kernel        | Request      | `SegmentRequest` |
| 28 | KernelSegment          | Kernel        | Response     | `SegmentResponse<TxKernel>` |

# Drawbacks
[drawbacks]: #drawbacks

Rewinding the bitmap accumulator to an old state is a relatively expensive operation.
The accumulator is not append-only and a rewind can cause large parts of it to be rewritten.
Implementations will most likely be required to (temporarily) store the old accumulator state either on disk or in memory, increasing the memory/storage requirements of running a node.

The amount of data required to be stored here is however much less than is required to support the txhashset.zip sync method.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The most obvious alternative to this proposal would be to keep the txhashset.zip sync method around indefinitely.
However, we do believe that this would be detrimental in the long term.
Outside of the disadvantages of this method mentioned in the [motivation section](#motivation), the method relies on the precise on-disk bit representation of MMRs.
We would be unable to improve upon the storage without breaking this sync method.
Furthermore, the zip sync method has been victim to a critical vulnerability before [\[7\]](#references).
While this vulnerability is long since patched, it is a testament to the complications that could arise in using zip archives as a synchronisation method.
Zip archives are a blunt tool and it would be unfortunate to have to permanently support it when we have access to a much more precise and efficient method.

We briefly considered adding a separate request for the output and bitmap roots at a certain height, as a block header only commits to the hash of the concatenated values: `H(output_root|bitmap_root)`.
While this would save a small amount of bandwidth compared to the current proposal, it would hurt our ability to precisely determine malicious peers.
If verification on a bitmap or output segment were to fail, it could be caused by either an invalid segment or by a collision in the roots.
We would be unable to distinguish between these two scenarios, forcing us to assume both peers are malicious.
However by including the other root hash directly into the segment response messages, we ensure all required data originates from the same peer.


# Future possibilities
[future-possibilities]: #future-possibilities

## Deprecate txhashet.zip
Upon connection, nodes signal whether they support zip sync and/or PIBD sync separately.
This allows us to deprecate and eventually remove support for the zip sync method in a future version.
It would remove the need for nodes to keep a copy of the most recent txhashset archive around, saving disk space.

Once the txhashset zip method has been removed, we could possibly relax a requirement on our segments.
Currently, for any pair of sibling leaves where only one of them is spent, we require _both_ of them to still be present in the segment.
This is because it is expected nodes will still support the zip sync method.
In this sync method, this property of sibling leaves has to hold due to historical implementation reasons.
Removing support for the zip sync could allow us to remove this requirement.

The precise details of the removal of support of the sync method and relaxation of segment requirement is considered to be out of the scope of this RFC.

## Header segments
Since nodes already maintain a block header MMR, we could re-use the existing segment infrastructure to introduce block header segments.
Although further investigation is required, this has the potential to speed up the header sync, which is the first step in the synchronisation process.

# References
[references]: #references
- \[1\] [Merkle mountain ranges](https://github.com/mimblewimble/grin/blob/master/doc/mmr.md)
- \[2\] [RFC 0009: enable faster sync](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0009-enable-faster-sync.md)
- \[3\] [Reference implementation - segments](https://github.com/mimblewimble/grin/pull/3453)
- \[4\] [Reference implementation - segmenter](https://github.com/mimblewimble/grin/pull/3482)
- \[5\] [Reference implementation - messages](https://github.com/mimblewimble/grin/pull/3470)
- \[6\] [Reference implementation - output bitmap message (de)ser](https://github.com/mimblewimble/grin/pull/3492)
- \[7\] [CVE-2019-9195](https://github.com/mimblewimble/grin-security/blob/master/CVEs/CVE-2019-9195.md)
