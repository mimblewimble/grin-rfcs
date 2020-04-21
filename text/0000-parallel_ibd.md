
- Title: parallel-ibd
- Authors: [Jasper van der Maarel](mailto:j@sper.dev), [Antioch Peverell](apeverell@protonmail.com)
- Start date: Apr 02, 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

This RFC describes improvements to the initial byte download (IBD) process for nodes. Previously, a new node would download all kernels, unspent outputs and rangeproofs in one file from a single peer. This data could only be verified after it had been fully downloaded. This presents both a performance bottleneck as well as a single point of failure. The changes in this RFC allow the data to be downloaded and verified in parallel batches from multiple peers, resulting in a speedup and a more robust syncing experience overall. This document describes the required p2p messages to enable this feature, as well as how nodes should verify the data in them. 

# Motivation
[motivation]: #motivation

TODO

# Community-level explanation
[community-level-explanation]: #community-level-explanation

A node joining the Grin network does not need the complete block history in order to fully verify the chain state. The block headers, the unspent output set and the complete kernel set are sufficient. The output and kernel sets are stored as leaves in a Merkle Mountain Range (MMR) and the block headers commit to the root of these trees. Prior to HF2, the block headers only committed to the output commitments and not their unspent/spent status. This meant that output and kernel data could only be verified after it had been completely downloaded, which forced nodes to download the full data from a single peer. The downsides of this are apparent: the download speed is bottlenecked by the bandwidth of the other peer and if that peer goes offline during download or provides malicious data (which can only be verified after the fact) the process had to be restarted from scratch from another peer.

However, due to a consensus change in HF2 the headers now also commit to the unspent/spent status. The output spent status is stored in a bitmap, which is split up in chunks of 1024 and stored in separate MMR. Block headers commit to the root of this MMR. This means that these chunks can be downloaded and verified independently, by providing two merkle proofs along the data that prove the inclusion of the unspent outputs and the output bitmap in the roots. It allows the data to be downloaded in parallel and verified as it comes in, which greatly improves the bootstrap time of a fresh node.

The consensus changes necessary to enable this feature were originally proposed in [RFC0009](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0009-enable-faster-sync.md), implemented in [grin#3108](https://github.com/mimblewimble/grin/pull/3108) and subsequently activated in HF2.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A bootstrapping node first downloads and verifies all block headers until the sync horizon point. From the final header it determines the number of leaves in both the output PMMR and kernel MMR. The order of the leaves in those trees are fixed and each leaf is uniquely identified by its position in the tree. The leaves will be divided up in a continuous series of chunks of size 1024. The final chunk is allowed to have less than 1024 elements. Each chunk is identified by an index, starting at 0. The node can request these chunks in any order from any up-to-date peer by using the `GetOutputChunk` and `GetKernelChunk` p2p messages described in the [following subsection](#p2p-messages).

### p2p messages
[p2p-messages]: #p2p-messages
The process in this RFC requires the introduction of 4 new peer-to-peer messages that follow request-response semantics:

**GetOutputChunk (21)**
- block hash (32 bytes)
- zero-based chunk index (8 bytes)

**OutputChunk (22)**
- block hash (32 bytes)
- zero-based chunk index (8 bytes)
- output bitmap (128 bytes)
- output count `N_output` (2 bytes)
- outputs (variable, `N_output * (X + 33 + 675)` bytes)
- output mmr hash count `N_oh` (2 bytes)
- output mmr hashes `(pos, hash)` (variable, `N_oh * (8 + 32)` bytes)
- output chunk merkle proof element count `N_om` (8 bytes)
- output chunk merkle proof (variable, `N_om * 32` bytes)
- bitmap merkle proof element count `N_bm` (8 bytes)
- bitmap merkle proof (variable, `N_bm * 32` bytes)

**GetKernelChunk (23)**
- block hash (32 bytes)
- zero-based chunk index (8 bytes)

**KernelChunk (24)**
- block hash (32 bytes)
- zero-based chunk index (8 bytes)
- kernel count `N_kernel` (2 bytes)
- kernels (variable, `N_kernel * Y` bytes)
- kernel chunk merkle proof element count `N_km` (8 bytes)
- kernel chunk merkle proof (variable, `N_km * 32` bytes)

### Chunk merkle proof
Whenever a node receives an output or kernel chunk, it needs to be able to verify that its contents are part of the MMR. This is usually done by providing a merkle proof, which in short is a list of hashes that, combined with the element hash is used to calculate the root. If it successfully reproduces the root, the proof is considered to be valid.

It would be very inefficient to provide a separate merkle proof for each leaf in the chunk. Therefore we create a single optimized merkle proof per chunk, based on two observations: firstly, a completely filled chunk (`2^10` elements) forms a full binary subtree in the MMR. This means it has a single root. Secondly, the final chunk can potentially be only partially filled (`<2^10` elements). If this is the case, each of the peaks in the chunk are direct peaks in the MMR.

As such, a chunk merkle proof is simply a list of hashes:
- Hashes of siblings along the path of subtree root to its peak (in case of a full chunk)
- Hashes of the peaks in the MMR, from right to left, excluding the peaks that can be calculated given the chunk contents. For efficiency reasons the peaks on the right side of our subtree are bagged together

### Sync process
After receiving an output chunk, it should be verified according to the following steps:

**Output chunk verification**
- Verify `N_output == cardinality(bitmap)`
- Fill unpruned leaves of output mmr subtree, calculate intermediary hashes where possible
- Fill missing intermediary hashes from the chunk data and calculate final missing hashes
- Use output chunk merkle proof to reconstruct `output_root`
- Use bitmap merkle proof to reconstruct `bitmap_root`
- Calculate `H(output_root|bitmap_root)` and compare with value in header
- Verify rangeproofs

If verification for this chunk passes, its contents should be stored somewhere. All unspent output commitments across all chunks need to eventually be summed up to a single commitment. The full output PMMR and bitmap MMR should be reconstructed as well. At which point in this process the (P)MMRs are constructed and the sum is calculated is left as an implementation choice.
   
The verification of kernel chunks is somewhat similar to output chunks, albeit somewhat simplified since kernels are never pruned:

**Kernel chunk verification**
- Fill leaves of kernel mmr subtree
- Use kernel chunk merkle proof to reconstruct `kernel_root` and compare with value in header
- Verify kernel signatures

If verification of a kernel chunk is successful, it should be stored. Similar to the output chunk, we need to eventually calculate a single sum across all chunks. In this case it is a sum of all kernel excess commitments. The subtree data should also be used to reconstruct the full kernel MMR. Another wrinkle is that we need to "replay" the kernel root at each height to prevent certain attacks. Whether to update these quantities when each chunk is accepted or only calculate them at once after all chunks have been received is once again left open as an implementation choice.

### Final check
Once all output and kernel chunks are downloaded and verified, the (P)MMRs are recreated and the appropriate sums have been calculated we need to perform a final check:
```
output_sum =?= kernel_sum + TOTAL_SUPPLY * H
```
Here `TOTAL_SUPPLY` is the total supply of Grins at the point of sync, which is equal to `(HEIGHT + 1)*COINBASE_REWARD`.

If all of this is successful, the node now has downloaded and verified the state up until the sync horizon point. Fully updating the chain state to the latest head involves downloading the full blocks after the sync horizon, verifying them and applying its contents consecutively, which is possible because all nodes are expected to store the full blocks past the compaction horizon.

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

- How expensive is reconstructing the output bitmap at arbitrary heights? Should we only support fixed heights, similar to the txhashset request?
- Do we need to explicitly signal for nodes to support these new p2p messages, or do we assume that any unupgraded will fall out of consensus due to the hard fork and eventually leave the network?

# Future possibilities
[future-possibilities]: #future-possibilities

TODO

# References
[references]: #references

TODO
