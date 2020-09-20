
- Title: pibd-messages
- Authors: [Jasper van der Maarel](mailto:j@sper.dev), [Antioch Peverell](mailto:apeverell@protonmail.com)
- Start date: September 14, 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

TODO 

# Motivation
[motivation]: #motivation

TODO

# Community-level explanation
[community-level-explanation]: #community-level-explanation

TODO

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

TODO

## MMR chunks
[mmr-chunks]: #mmr-chunks
We define a (P)MMR **chunk** as a set of `N = 2^b` consecutive leaves, along with the necessary data to reconstruct the subtree root and data to verify membership of the subtree in the original MMR.
Each chunk is consecutive with the previous and is identified by the tuple `(H, b, i)` where `H` is the block hash and `i` is a zero-based chunk index: chunk `(H, b, i)` contains the non-pruned leaves with leave position in the interval `[i*N, (i+1)*N)`.
The final chunk is allowed to have less than `N` elements because the full MMR does not necessarily contain a multiple of `N` leaves.

A chunk contains, besides the identifier tuple and the unpruned leaf data, a list of pruned hashes of the leaves and a chunk MMR proof.
After receiving a chunk, clients should verify if its contents are contained in the MMR by performing the following two steps: 
1) calculate the root of the subtree (using the leaf data and pruned hashes) and 
2) reconstruct the full MMR root using the subtree root and the [chunk merkle proof](#chunk-merkle-proof) and compare it to the root in the block header.

Implementations are free to choose a chunk size by setting `b` within a certain range in their requests.
This allowed range is set differently depending on the MMR contents.
The following table has concrete values for the different MMR types and the allowed range of size.

| MMR Type      | Prunable? | Leaf size (B) | `b` Range   |
|---------------|-----------|---------------|-------------|
| Output bitmap | No        | 128           | TBD         |
| Outputs       | Yes       | 34            | TBD         |
| Range proofs  | Yes       | 675           | TBD         |
| Kernels       | No        | ~114          | TBD         |

### Chunk merkle proof
[chunk-merkle-proof]: #chunk-merkle-proof
Whenever a node receives an MMR chunk, it needs to be able to verify that its contents are in fact part of the MMR.
This is usually done by providing a merkle proof, which in short is a list of hashes that, combined with a leaf hash, is used to calculate the root.
If it successfully reproduces the root, the proof is considered to be valid.

It would be very inefficient to provide a separate merkle proof for each leaf in the chunk.
Therefore we create a single optimized merkle proof per chunk, based on two observations: firstly, a full chunk (`2^b` elements) forms a full binary subtree in the MMR. 
This means it has a single root. 
Secondly, the final chunk is potentially only partially filled (number of elements is not a power of 2). 
If this is the case, each of the peaks in the chunk are direct peaks in the MMR.

As such, a chunk merkle proof is simply a list of hashes:
- Hashes of siblings along the path of subtree root to its peak (in case of a full chunk)
- Hashes of the peaks in the MMR, from right to left, excluding the peaks that can be calculated given the chunk contents. 
For efficiency reasons the peaks on the right side of our subtree are bagged together.

### p2p messages
[p2p-messages]: #p2p-messages

This RFC introduces 8 new peer-to-peer messages that follow request-response semantics: 2 for each MMR type.

Each request contains the chunk identifier tuplet `(H, b, i)`:
- block hash `H` (32 bytes)
- 2log of chunk size: `b` (1 byte)
- zero-based chunk index `i` (8 bytes)

The response is a chunk as explained in the [previous section](#mmr-chunks):
- block hash `H` (32 bytes)
- 2log of chunk size: `b` (1 byte)
- zero-based chunk index `i` (8 bytes)
- vector of pruned hashes (prunable MMR: `8 + 32*N_ph` bytes, non-prunable MMR: 0 bytes)
- vector of `(pos, leave data)` (`8 + (8 + X)*N_l` where `X` is the size of a leaf)
- chunk merkle proof: vector of hashes (`8 + 32*N_mp` bytes)

The new p2p message are as follows:

| ID | Name                 | MMR type      | Message type |
|----|----------------------|---------------|--------------|
| 21 | GetOutputBitmapChunk | Output bitmap | Request      |
| 22 | OutputBitmapChunk    | Output bitmap | Response     |
| 23 | GetOutputChunk       | Output        | Request      |
| 24 | OutputChunk          | Output        | Response     |
| 25 | GetRangeProofChunk   | Range proof   | Request      |
| 26 | RangeProofChunk      | Range proof   | Response     |
| 27 | GetKernelChunk       | Kernel        | Request      |
| 28 | KernelChunk          | Kernel        | Response     |

# Drawbacks
[drawbacks]: #drawbacks

TODO

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

TODO

# Prior art
[prior-art]: #prior-art

TODO

# Unresolved questions
[unresolved-questions]: #unresolved-questions

TODO

# Future possibilities
[future-possibilities]: #future-possibilities

TODO

# References
[references]: #references

TODO
