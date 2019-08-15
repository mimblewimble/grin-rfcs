
- Title: variable-size-kernels
- Authors: [Antioch Peverell](mailto:apeverell@protonmail.com)
- Start date : Aug 13, 2019
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

We can reduce the amount of data used to serialize transaction kernel in binary format by including only the data applicable for each kernel feature variant. Specifically we only need a lock height on height locked kernels and coinbase kernels have no associated fee. Each kernel feature variant will serialize to a known number of bytes but this size will not be consistent across different feature variants.

# Motivation
[motivation]: #motivation

We currently include both the fee and the lock_height on _every_ kernel in binary serialization format. We store a fee of 0 on coinbase kernels and a lock_height of 0 on plain and coinbase kernels. Kernels are never pruned and exist forever so this overhead is relatively expensive. By only serializing data strictly necessary for each kernel feature variant we reduce data storage and transmission costs. This change will also introduce the flexibility necessary to more easily introduce future kernel variants, for example the proposed [relative kernels][0].

# Community-level explanation
[community-level-explanation]: #community-level-explanation

Each transaction kernel feature variant may have some associated data. For example height locked kernels include an associated lock height and plain kernels have an associated fee. Coinbase kernels have no associated fee nor lock height. Binary serialization of each kernel feature variant produces a fixed size in bytes but this size may differ across different feature variants. This allows kernels to be serialized efficiently in terms of size in bytes and also provides flexibility to introduce new kernel variants that have additional associated data in the future.

To illustrate the benfit of saving a few bytes, we observe that 8 bytes are used to serialize the transaction fee. By not including 8 bytes for the fee on coinbase kernels we save 8,000,000 bytes of unneccesary data over one million coinbase kernels.

These changes affect serialization/deserialization of transaction kernels and these are included in several p2p messages, including transactions, full blocks and compact blocks. We need to be careful with the rollout of these changes to ensure backward compatibility across nodes on the network. The kernel MMR is received as part of the initial state sync so care must be taken here to ensure all nodes can continue to sync successfully when joining the network.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

In protocol version 1 all transaction kernels are serialized using the same structure, regardless of kernel variant. All kernels include 8 bytes for the fee and 8 bytes for the lock_height, even if unused.

```
features (1 byte) | fee (8 bytes) | lock_height (8 bytes) | excess (32 bytes) | signature (64 bytes)

00 | 00 00 00 00 01 f7 8a 40 | 00 00 00 00 00 00 00 00 | 08 b1 ... 22 d8 | 33 11 ... b9 69
```

All kernels contain a feature byte to determine the variant. Kernels always include an excess commitment and signature, 32 bytes and 64 bytes respectively. The proposal is to have variant specific data follow the feature byte.

The initial feature byte would determine the number of subsequent bytes to read for variant specific data.

```
00 (plain): following 8 bytes for the fee.
01 (coinbase): no additional bytes.
02 (height locked): following 8 bytes for the fee and additional 8 bytes for the lock height.
```

This would always be followed by a fixed 32 bytes for the excess commitment and 64 bytes for the kernel signature.

----

Proposed plain kernel (protocol version 2) -

```
features (1 byte) | fee (8 bytes) | excess (32 bytes) | signature (64 bytes)

00 | 00 00 00 00 01 f7 8a 40 | 08 b1 ... 22 d8 | 33 11 ... b9 69
```

Proposed coinbase kernel (protocol version 2) -

```
features (1 byte) | excess (32 bytes) | signature (64 bytes)

01 | 08 b2 ... 15 36 | 08 14 ... 98 96
```

Proposed height locked kernel (protocol version 2) -

```
features (1 byte) | fee (8 bytes) | lock_height (8 bytes) | excess (32 bytes) | signature (64 bytes)

02 | 00 00 00 00 00 6a cf c0 | 00 00 00 00 00 00 04 14 | 09 4d ... bb 9a | 09 c7 ... bd 54
```

----

#### Backward Compatibility (Protocol Version Support)

[wip] Discuss compatibility between nodes across protocol version 1 and protocol version 2. Both for transaction and block p2p messages and the less flexible heavyweight txhashset.zip

How do we structure compatibility rules here to ensure nodes can still successfully communicate if bytes appear to be missing etc.

We also need to account for in-place migration. Full blocks are stored in the local database.

Each node maintains a local kernel MMR - but this is also used to generate txhashset.zip which is sent to other nodes, so not limited to local node.

----

#### Wallet Compatibility

Interactive transaction building involves passing a transaction "slate" around. This includes a json serialized representation of the transaction.

To minimize compatibility issues between walets we are not planning to change this json format or structure. Transaction kernels will continue to be translated to a single consistent json format, always including values for both fee and lock_height, regardless of feature variant. For example plain kernels will always have lock_height of 0 and coinbase kernels will always have 0 fee. This is consistent with the current "v2" slate.

```
"kernels": [
  {
	"features": "Plain",
	"fee": "7000000",
	"lock_height": "0",
	"excess": "08b1...22d8",
	"excess_sig": "3311...b969"
  }
]
```

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Future possibilities
[future-possibilities]: #future-possibilities

One motivation for "variable size kernels" is to allow for new kernel variants to be introduced in the future. One concrete exampe of this is the proposal for [relative kernels][0]. Kernels of this feature variant will have an associated reference to a prior kernel and the existing "fixed size" binary serialization format does not provide the flexibility to include this additional data.

# References
[references]: #references

* [RFC 0000-relative-kernels][0] [fix link once RFC merged]

[0]: https://github.com/mimblewimble/grin-rfcs/blob/master/text/0000-relative-kernels.md
