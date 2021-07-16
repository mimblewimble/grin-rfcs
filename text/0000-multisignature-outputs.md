
- Title: [multisignature-outputs]
- Authors: [Gene Ferneau](mailto:gene@ferneau.link)
- Start date: July 16, 2021
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#85](https://github.com/mimblewimble/grin-rfcs/pull/85)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

## Summary
[summary]: #summary

Multisignature outputs allow for shared ownership over a single output. Applications range from escrow services, multisig wallets, atomic swaps, and more. This RFC specifies 2-of-2 multisignature outputs, but can easily be extended to N-ofN multisignature outputs. M-of-N multisignature outputs (threshold multisignature) is left to future work.

## Motivation
[motivation]: #motivation

Multisignature outputs allow for shared ownership of a single output, and applications that require multiple parties to spend a single output. The most immediate application is [atomic swaps](https://github.com/mimblewimble/grin-rfcs/83). Other possibilities extend to escrow services, multisignature wallets, and more.

## Community-level explanation
[community-level-explanation]: #community-level-explanation

Multisignature outputs are built in a somewhat similar way to normal outputs, but require more communication rounds to build the shared commitment and rangeproof.

Currently, the transaction building requires five communication rounds. A number of changes to Slatepack formatting are required, and a few protocols for deriving shared secrets among participants.

A new Slatepack version is required (version 5). Slate version 5 includes changes for multisignature outputs and atomic swaps, to minimize code churn. Community members don't have to worry about these changes.

At the time of writing, the implementation supports 2-of-2 multisignature outputs, which can be extended to N-of-N multisignature outputs. M-of-N (threshold) multisignatures require fundamental changes in the cryptographic implementation.

The first two communication rounds (`init_tx` and `receive_tx`) are largely the same, with some additional arguments. An additional `process_multisig_tx` and `finalize_tx` round are added for multiparty bulletproof construction.

The total rounds are:

- `init_tx`
- `receive_tx`
- `process_multisig_tx`
- `finalize_tx` (receiver)
- `finalize_tx` (initiator)

The `process_multisig_tx` and `finalize_tx (receiver)` transactions are needed for multiparty bulletproof construction, and safely signing the transaction kernel.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Multisignature transactions require a number of changes to slate formatting, exposing cryptographic primitives from libsecp256k1, transaction building in the node and wallet implementation, and wallet commands exposed to the user.

### Slate version 5
[slate-version]: #slate-version

Versioned slates allow for the incremental addition of new features to Grin transaction building. A new slate version 5 is proposed to include changes for multisignature outputs and atomic swaps.

Originally slate versions corresponded with hard forks in the Grin protocol, but there are no longer any scheduled hard forks. This means a soft fork is required to implement these changes.

- Add slate states Multisig1 (7), Multisig2 (8), Multisig3 (9), and Multisig4 (10) to indicate multsignature communication rounds
  - The numbers represent the unsigned integer representation when the slate state is serialized
- New optional fields for transmitting multiparty bulletproof data
  - part_commit - Pedersen commitment
  - tau_one - Secp256k1 public key
  - tau_two - Secp256k1 public key
  - tau_x - Secp256k1 private key
- Encoding the presence of optional fields in the optional flag (currently 0 or 1)
  - 1-bit set indicates partial signature (`part`) present
  - 2-bit set indicates atomic public key (`atomic`) present
  - 4-bit set indicates partial commit (`part_commit`) present
  - 8-bit set indicates Tau X key (`tau_x`) present
  - 16-bit set indicates Tau One key (`tau_one`) present
  - 32-bit set indicates Tau Two key (`tau_two`) present
- `SigWrap` is serialized in the following order
  - length (number of ParticipantData entries)
  - each ParticipantData entry
- `ParticipantData` is serialized in the following order
  - optional flag (unsigned 8-bit integer)
  - blinding factor (Secp256k1 public key, compressed)
  - nonce (Secp256k1 public key, compressed)
  - atomic public key (Secp256k1 public key, compressed) (if present)
  - partial signature (Secp256k1 signature) (if present)
  - partial commit (Pedersen commitment) (if present)
  - tau x key (Secp256k1 secret key) (if present)
  - tau one key (Secp256k1 public key, compressed) (if present)
  - tau two key (Secp256k1 public key, compressed) (if present)
- ProtocolVersion bumped to 5
- Add optional `multisig_key_id` field (Secp256k1 Identifier) to base slate fields
  - `multisig_key_id` is serialized after all other fields
  - `multisig_key_id` is serialized as:
    - an unsigned 8-bit integer (1=present, 0=absent)
    - 19-byte serialized Identifier (if present)

The `multisig_key_id` is a unique identifier for the multisig output, and is derived from the Slate UUID. It is used in slates spending the multisig output, so that the output can be selected by all multsignature parties. The `multisig_key_id` is not included during multsignature output transaction building, since both parties can derive the ID independently.

### Multisignature Key ID
[multisig-key-id]: #multisig-key-id

To accurately build, select, and spend a multisignature output, it is necessary to derive and communicate a multisignature key ID.

To derive the multsig key id during multsig transaction building, each party hashes a domain separation string, the Slate ID, and transaction amount. Since the Slate ID is a UUID (universal *unique* identifier), each multisignature key ID should also be unique. The hash used is Blake2b for its ability to output 16-byte hashes, and use other places in the Grin implementation.

The derivation function:

```
id_hash = Blake2b("multisig_id" || SlateIDBytes || AmountBytesBE, 16)
```

where `SlateIDBytes` is the serialized byte representation of the Slate UUID, and `AmountBytesBE` is the 64-bit unsigned integer amount in 8-byte big-endian form.

The multisig key ID is a BIP32-style identifier, constructed from the hash output (using python slice notation):

```
Identifier(depth: 4, id_hash[0:4], id_hash[4:8], id_hash[8:12], id_hash[12:16])
```

### Common nonce
[common-nonce]: #common-nonce

To build a multiparty bulletproof, a common nonce is required over the shared commitment to the transaction amount.

The common nonce is derived using a Diffie-Hellman exchange of the public nonces of the participants.

The result of the Diffie-Hellman exchange is hashed together with a domain separation string `multisig_common_nonce` using the SHA3-256 hashing algorithm:

```
// For party a:
shared_key_in = Diffie-Hellman(sec_nonce_a, pub_nonce_b)
common_nonce = SecretKey(Sha3_256("multisig_common_nonce" || shared_key_in_bytes))

// For party b:
shared_key_in = Diffie-Hellman(sec_nonce_b, pub_nonce_a)
common_nonce = SecretKey(Sha3_256("multisig_common_nonce" || shared_key_in_bytes))
```

where `shared_key_in_bytes` is the byte representation of the serialized Secp256k1 secret key.

SHA3-256 is used for its security, 32-byte output, and use in other parts of the Grin implementation (e.g. slatepack addresses).

### Multiparty bulletproofs

In this section, building multiparty bulletproofs is described. The process requires mutliple communication rounds to exchange the partial keys used for constructing the final proof.

The protocol follows the specification originally proposed in the bulletproofs white paper by Buenz et al. [\[1\]](#references) (Ss. 4.5 "A Simple MPC Protocol for Bulletproofs"), and the implementation in the Grin fork of `secp256k1-zkp` [\[2\]](#references).

#### init_send_tx

In the first round, the initiator commits to value of the transaction using their `multisig_key`. The `multisig_key_id` is generated according to the previous section.

```
(multisig_key_a_pub, multisig_key_a_sec) = a.keychain().derive_key(multisig_key_id)

part_commit_a = PedersenCommit(value, multisig_key_a_sec)
```

The partial commit is added to the slate, and sent to the receiver.

#### receive_tx

The receiver extracts the initiator's partial commit from the slate, and computes their own partial commit:

```
(multisig_key_b_pub, multisig_key_b_sec) = b.keychain().derive_key(multisig_key_id)

part_commit_b = PedersenCommit(value, multisig_key_b_sec)
```

The receiver then adds the Pedersen commitments together to get the `commit_sum`. The `commit_sum` is used for the remainder of the multiparty bulletproof building protocol, and is the commitment for the finalized bulletproof.

```
commit_sum = secp.commit_sum([part_commit_a, part_commit_b], [])
```

Now, the receiver can perform the first step of building the multiparty bulletproof, and generate their `tau_one` and `tau_two` public keys:

```
tau_one_b = Some(PublicKey::new())
tau_two_b = Some(PublicKey::new())
// Pseudo-code based on the Rust interface
create_multisig(
    keychain,
    ProofBuilder(keychain),
    amount,
    multisig_key_id,
    RegularSwitchCommitment,
    common_nonce,
    NullTauX,
    // Mutable reference or pointer to return the Tau One Secp256k1 public key
    tau_one_b,
    // Mutable reference or pointer to return the Tau Two Secp256k1 public key
    tau_two_b,
    [commit_sum],
    1 /* Round 1 */,
    NoExtraData,
)
```

For C code a similar interface exists in the Grin fork of libsecp256k1-zkp [\[2\]](#references).

The receiver adds the output to the transaction as a multisig output type, with the `commit_sum` commitment and a dummy rangeproof. The value for the proof is added at the end of the protocol.

The receiver adds their `part_commit_b` to the slate, along with `tau_one_b` and `tau_two_b`, and sends the slate back to the initiator.

In the Rust implementation, the multisig output is saved to the local database using the `parent_key_id` as the `root_key_id`, and the `multisig_key_id` as the `key_id`. This is somewhat of an implementation detail, but it is important to save the multisig output for both parties so it can be retrieved for spending. Other implementations will have their own methods for storing transactions, but using the `multisig_key_id` to index the output in some way is recommended.

#### process_multisig_tx

In the third round, the initiator extracts the receiver's `partial_commit_b` to reconstruct the `commit_sum`. The initiator also gets the `tau_one_b` and `tau_two_b` public keys to construct `tau_one_sum` and `tau_two_sum` later.

First, the initiator must create their own tau public keys:

```
tau_one_a = Some(PublicKey::new())
tau_two_a = Some(PublicKey::new())
// Pseudo-code based on the Rust interface
create_multisig(
    keychain,
    ProofBuilder(keychain),
    amount,
    multisig_key_id,
    RegularSwitchCommitment,
    common_nonce,
    NullTauX,
    // Mutable reference or pointer to return the Tau One Secp256k1 public key
    tau_one_a,
    // Mutable reference or pointer to return the Tau Two Secp256k1 public key
    tau_two_a,
    [commit_sum],
    1 /* Round 1 */,
    NoExtraData,
)
```

The initiator can now sum together the keys with the receiver's tau public keys:

```
tau_one_sum = tau_one_a + tau_one_b
tau_two_sum = tau_two_a + tau_two_b
```

The tau sum keys will be used for the remainder of the protocol. The initiator can now produce their part of the `tau_x` secret key:

```
tau_x_a = Some(SecretKey::new());
// Pseudo-code based on the Rust interface
create_multisig(
    keychain,
    ProofBuilder(keychain),
    amount,
    multisig_key_id,
    RegularSwitchCommitment,
    common_nonce,
    // Mutable reference or pointer to return the Tau X Secp256k1 secret key
    tau_x_a,
    // Mutable reference or pointer to the Tau One Secp256k1 public key
    tau_one_sum,
    // Mutable reference or pointer to the Tau Two Secp256k1 public key
    tau_two_sum,
    [commit_sum],
    2 /* Round 2 */,
    NoExtraData,
)
```

The initiator adds their partial tau keys to the slate, and stores the sum keys locally:

```
slate.tau_x = tau_x_a
slate.tau_one = tau_one_a
slate.tau_two = tau_two_a

store(tau_one_sum)
store(tau_two_sum)
```

#### finalize_tx (receiver)

There are two final rounds for the multiparty bulletproof protocol, one for the initiator and receiver. The receiver goes first, and the initiator completes the multiparty bulletproof.

In these final rounds, the parties can also add their partial kernel signatures to the transaction. This allows for the initiator to post the transaction after their finalization round.

Receiver extracts `tau_one_a`, `tau_two_a`, and `tau_x_a` from the slate, and reconstructs `tau_one_sum` and `tau_two_sum`:

```
tau_one_sum = tau_one_b + tau_one_a
tau_two_sum = tau_two_b + tau_two_a
```

Now, the receiver creates their own `tau_x` secret key:

```
tau_x_b = Some(SecretKey::new());
// Pseudo-code based on the Rust interface
create_multisig(
    keychain,
    ProofBuilder(keychain),
    amount,
    multisig_key_id,
    RegularSwitchCommitment,
    common_nonce,
    // Mutable reference or pointer to return the Tau X Secp256k1 secret key
    tau_x_b,
    // Mutable reference or pointer to the Tau One Secp256k1 public key
    tau_one_sum,
    // Mutable reference or pointer to the Tau Two Secp256k1 public key
    tau_two_sum,
    [commit_sum],
    2 /* Round 2 */,
    NoExtraData,
)
```

The receiver calculates `tau_x_sum`, and adds their partial `tau_x_b` to the slate. The receiver also creates their partial kernel signature, and adds it to the slate.

```
slate.tau_x = tau_x_b
// ensure that the multisig output is included for the receiver
slate.part = calculate_partial_kernel_signature()
```

Now the slate is sent back to the initiator for the final round.

#### finalize_tx (initiator)

The initiator extracts `tau_x_b` from the slate, and reconstructs `tau_x_sum`. This key is used along with the other tau sum keys to compute the final bulletproof:

```
tau_x_sum = tau_x_a + tau_x_b
// Pseudo-code based on the Rust interface
create_multisig(
    keychain,
    ProofBuilder(keychain),
    amount,
    multisig_key_id,
    RegularSwitchCommitment,
    common_nonce,
    // Mutable reference or pointer to return the Tau X Secp256k1 secret key
    tau_x_sum,
    // Mutable reference or pointer to the Tau One Secp256k1 public key
    tau_one_sum,
    // Mutable reference or pointer to the Tau Two Secp256k1 public key
    tau_two_sum,
    [commit_sum],
    0 /* Final Round */,
    NoExtraData,
)
```

Note that the final round is `0` and not `3`, this is the same in the Rust and C interfaces. In the Rust interface, this final call will return the finalized bulletproof.

Together with the `commit_sum` commitment, the multisig output is now complete! The initiator should store the multisig output, in a similar way to the receiver.

After calculating their partial kernel signature, and adding it to the receiver's partial kernel signature, the initiator can post the transaction.

Ensure that the multisig output is added to the initiators list of outputs when calculating the partial kernel signature.

The receiver can extract the final bulletproof from the posted transaction, and update their stored output.

## Drawbacks
[drawbacks]: #drawbacks

Multisignature outputs require two additional communication rounds, which makes end-user UX worse. During synchronous transaction building, there will be no noticeable difference. However, fallback/manual transactions will be more complex.

Multsignature outputs require a soft fork for changes to the slate version, and transaction building. The new slate version is backwards compatible with old software, in the sense that new fields will be ignored. However, it is not possible to use the new features introduced by the new slate version with old software.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- There are no known alternatives using MuSig style multisignatures
- FROST threshold signatures [\[3\]](#references) may be possible in Mimblewimble-style blockchains, but requires further research
  - May allow for M-of-N multisignatures
  - Unclear if it is compatible with Bulletproofs used in Mimblewimble

## Prior art
[prior-art]: #prior-art

Prior to this implementation, no multisignature support was present in Grin implementations. The underlying cryptographic primitives existed, but were not glued together into transaction building.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- Is FROST possible on Mimblewimble-style chains?
- Are there more efficient designs for multiparty Bulletproof construction?
- Can the communication rounds be reduced to improve UX?

## Future possibilities
[future-possibilities]: #future-possibilities

More research may reveal how to incorporate M-of-N multisignature support. This would be a big win, and open up many more possibilities for applications like decentralised recovery, arbitrage, etc.

Extending the current protocol to support N-of-N for more than two parties could also be worth researching. Groups that require all members to sign off on a transaction could benefit from larger N-of-N multisignature support.

## References
[references]: #references

- [1](https://eprint.iacr.org/2017/1066.pdf)
- [2](https://github.com/mimblewimble/secp256k1-zkp/blob/master/include/secp256k1_bulletproofs.h#L137-L182)
- [3](https://eprint.iacr.org/2020/852.pdf)
