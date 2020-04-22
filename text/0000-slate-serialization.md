- Title: slate-serialization
- Authors: [joltz](mailto:joltz@protonmail.com)
- Start date: March 31 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

This RFC standardizes transaction slate serialization for Grin. It should require no new changes to the current `grin-wallet` implementation and is meant to serve as a reference to maintain compatibility when serializing and deserializing Grin slates across a variety of implementations and wallets.

This RFC only describes _how slate blobs are serialized_ for transport. The specific details for what the _contents of a compatible slate blob_ are can be found in the [Compact Slates RFC](#). The specific details for standard _methods of slate blob transport_ can be found in a future [Standardized Transactions RFC](#).

# Motivation
[motivation]: #motivation

A standard for slate serialization ensures that existing and future Grin wallets can exchange Grin slates with any other Grin wallet without worrying about inconsistencies when serializing and deserializing transaction slates to and from their native application objects.

This RFC should provide clarity and support to wallet developers so that they can be confident in maintaining compatibility with other wallets. This is critical for Grin because today transactions require interactivity and by extension compatibility between wallets.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

To exchange Grin, users must exchange transaction slates in one form or another. A transaction slate is a blob containing the necessary data to be included at each step of the transaction building process. This RFC specifies a standard for wallets to follow to ensure that transaction slate blobs sent and received are universally compatible across all wallets. To accomplish this it is necessary to specify the technique used to serialize or encode the slate data for Grin transactions.

Currently Grin supports two methods for serializing slates: JSON and binary\*. For transaction methods that utilize direct wallet to wallet communication like Tor, Grin slates are serialized as JSON for transport. For transaction methods that utilize out-of-band slate exchange like file or armored-text, a binary serialization technique is used*.

Details of each serialization method can be found in the next section. All wallets should support both serialization techniques to ensure compatibility with other wallets.

Wallet developers that are building with `libwallet` have this layer abstracted away and do not need to be as concerned with the exact details of slate serialization.

_\* Binary serialization is still a WIP_

## Example

In this example, a `grin-wallet` user will send Grin to a user of an unknown wallet using the Tor transaction method and JSON-RPC with special attention paid to the slate serialization.

1. The sender obtains the receivers .onion address and prepares a transaction:
  - `grin-wallet` prepares to build a new transaction using its native application objects
  - `grin-wallet` serializes the native slate application object as JSON once the slate building operations have completed
  - `grin-wallet` sends the transaction slate as JSON to the receiver's .onion address via JSON-RPC
2. The recipients wallet receives a JSON-RPC message with the JSON encoded slate as the payload
  - The unknown wallet deserializes the received JSON into its native application object used for building transaction slates.
  - The unknown wallet prepares additional data and signatures needed to build the return slate
  - The unknown wallet serializes its native slate application object into JSON conforming to established slate standards
  - The unknown wallet returns the return slate encoded as JSON in the message payload sent to the sender
3. Sender receives the return slate as JSON in the message payload
  - `grin-wallet` deserializes the JSON into a native application object for processing
  - `grin-wallet` completes necessary signatures and broadcasts the required transaction data to a node to be included in a block

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This section describes the details for how Grin slate blobs are serialized. For the specification of the contents of the slate blob itself see the [Compact Slates RFC](#). For the specification on the transport of slate blobs see the [Standardized Transactions RFC](#).

The serialization methods described in this RFC are already implemented in `grin-wallet` so this reference-level explanation will focus on slate serialization generically enough to be implemented in any language.

Grin slates can be serialized with two methods, depending on the transaction method used to exchange the slate. Wallet-to-wallet interactions exchange slate data serialized as JSON. Out-of-band interactions exchange slate data serialized as binary via file or armored-text.

Note that the actual mode of exchange of the JSON encoded slate (e.g. JSON-RPC) is out of scope of this RFC as currently written. These details should be specified in a future [Standardized Transactions RFC](#).

## JSON Serialization

[JSON](https://www.json.org/json-en.html) is a commonly used encoding format to represent particular application objects as universal human-readable blobs. Grin slates are serialized as JSON for direct wallet-to-wallet transport methods like Tor.

The reference library used for JSON serialization and deserialization in `grin-wallet` is [`serde_json`](https://docs.rs/serde_json/1.0.51/serde_json/index.html).

An example slate application object representation in `grin-wallet`:
```
EXAMPLE
```

An example slate representation as serialized with JSON:
```
EXAMPLE
```

JSON is just a key/value representation of the slate data encoded as a universally exchangeable blob.

## Binary Serialization

Binary is a low-level representation of data. Transaction slate application objects in Grin are serialized as binary blobs for asynchronous transport like file or armored-text. Slates serialized as binary blobs may take less data to represent the slate and so can be a beneficial serialization method when manually transporting slates out-of-band where size can be a limiting factor.

The specifics of serializing slates as binary are not yet finalized and can be tracked in the [Binary Slates PR](https://github.com/mimblewimble/grin-wallet/pull/385). Notably the first-order slate object is still JSON.

Following the above example, here is the same slate object binary serialization representation:
```
EXAMPLE
```

## Deserialization

Deserialization from JSON or binary blob to the native application object is left as an exercise to each implementation. `grin-wallet` uses [`serde_json`](https://docs.rs/serde_json/1.0.51/serde_json/index.html).

# Drawbacks
[drawbacks]: #drawbacks

- By supporting a binary serialization method in addition to the existing JSON serialization method, more complexity is required for developers building wallets on alternative implementations.
- Developers may not want to support all transaction methods if it requires supporting multiple serialization methods (e.g. a wallet developer may not want to update code to support a new file binary serialization and so might deprecate file use completely rather than writing the new code)

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

There are many alternatives that could be used for slate serialization. The serialization methods of this RFC were chosen for a few reasons:
  - `grin-wallet` already uses `JSON-RPC` to exchange slate data wallet to wallet and changing this is a nontrivial amount of work
  - Available solutions like MessagePack/CapnProto/ProtoBuf and derivatives don't seem to give enough benefits to justify the added complexity compared to a minimal custom serialization leveraging existing code
  - SaltPack serialization adds support for encryption and future extendability immediately at the cost of a nontrivial amount of complexity
  - Complexity beyond basic JSON and binary serialization do not seem to provide enough efficiency improvements to make an order of magnitude difference in the ability to improve developer or user experiences

By not specifying a standard the Grin ecosystem risks slate incompatibilities between wallets and services resulting in frustrated developers and confused users.

# Prior art
[prior-art]: #prior-art

There has been some [related discussion]((https://rfc.tari.com/RFC-0171_MessageSerialisation.html) in projects like Tari. However most of this work is focused around building and serializing message payloads in general.

The scope of this RFC is focused around specifically serializing slate payloads, which are somewhat unique to Grin. Due to the current necessity of interaction when transaction building and the nature of manual human handling of transaction slates, it makes sense to consider serialization techniques for slate payloads as distinct from serialization techniques for message payloads in general.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should this RFC represent a serialization specification for Grin in general or only `grin-wallet`?

- Does it seem odd that JSON is used for the in-band machine handling of slates and binary is used for out-of-band human handling?

- Will binary serialization reduce the size of serialized slates enough to justify the added complexity to support two slate serialization techniques?

- Does it make sense to include binary serialization in the scope of armored/QR slates instead if JSON remains the first-order object when serializing and it would only be used for those methods?

- Is it reasonable to ask developers of alternative implementations to support multiple serialization methods for slates?

# Future possibilities
[future-possibilities]: #future-possibilities

- This RFC is required to support a standard for armored slates

# References
[references]: #references

https://msgpack.org

https://github.com/msgpack/msgpack/blob/master/spec.md

https://en.wikipedia.org/wiki/Comparison_of_data-serialization_formats

https://saltpack.org

https://github.com/mimblewimble/grin/issues/39

https://rfc.tari.com/RFC-0171_MessageSerialisation.html

http://zguide.zeromq.org/py:chapter7#Serialization-Libraries
