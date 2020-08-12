
- Title: Eliminate Finalize Step
- Authors: [David Burkett](mailto:davidburkett38@gmail.com)
- Start date: Aug 09, 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

Use one-time addresses which include a pre-generated public excess and public nonce in order to eliminate the 3rd step in transaction building (finalize) [1].

# Motivation
[motivation]: #motivation

This change simplifies transaction building by effectively reducing the number of steps required to build a transaction. This is seen most clearly in the case of mobile-to-mobile transacting. Before, the sender would have to scan a QR code from the receiver, then the receiver would scan a QR code from the sender, and then the sender would once again scan a QR code from the receiver. That's 3 QR scans total, and an awkward and confusing process overall. After this change, the final QR is no longer required. It just requires 2 scans: sender <- receiver, receiver <- sender.

By reducing the number of steps required, we're also reducing the number of states a transaction can be in. This simplifies transaction management logic, decreases the complexity of debugging, and makes for a more shallow learning curve for Grin users.

Before, a transaction could be:
* Sent, not received
* Received, not finalized
* Finalized, not confirmed

After, a transaction can be:
* Sent, not received
* Received, not confirmed

Lastly, there have been a number of cases where users have tried to receive funds, but after receiving the slate and returning it to the sender, something goes wrong when finalizing or broadcasting. The receiver gets no indication that a problem occurred other than the transaction never showing up on the blockchain. Sometimes, there's an issue with the slate and the sender is unable to finalize, while other times the transaction just never made it to a block and needs rebroadcast. This change provides the receiver with a greater ability to diagnose and resolve these problems on her own, which pushes the burden to the party who is most motivated to resolve the issue.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

## Prior to this RFC

### Sender initiated (regular workflow)
1. Receiver shares a `grin1` slatepack address with sender.
2. Sender creates and sends partial slate to receiver's slatepack address.
3. Receiver produces a response and returns to sender.
4. Sender finalizes and broadcasts to the network.

### Receiver initiated (invoice workflow)
1. Sender shares a `grin1` slatepack address with sender.
2. Receiver creates a payment request and partial slate and sends to sender.
3. Sender approves the request, produces a response and returns to receiver.
4. Receiver finalizes and broadcasts to the network.

## Flow as per this RFC

### Sender initiated (regular workflow)
1. Receiver shares a one-time address with sender.
2. Sender creates a partial slate and shares with receiver.
3. Receiver finalizes and broadcasts transaction to the network.

### Receiver initiated (invoice workflow)
1. Receiver shares a one-time address, amount, and optional memo, expiration, etc.
2. Sender approves the request, creates a partial slate and shares with receiver.
3. Receiver finalizes and broadcasts to the network.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Address Generation

As of the introduction of slatepacks, we already have a standardized system for generating addresses [4]. To support eliminating the finalize step, we must extend these addresses to include a public excess and a public nonce, and a signature using the `ed25519_key` committing to ownership of the excess and nonce. The new addresses are for one-time use only.

The address format will be `version | ed25519_key | public_excess | public_nonce | signature` bech32 encoded using `grin` as the HRC and `1` as the version. They are easily distinguishable from the existing slatepack addresses due to their additional length (1 additional byte for version, 33 additional bytes for `public_excess`, 33 additional bytes for `public_nonce`, and 64 additional bytes for `signature`).

### Example

#### Data to encode
* version: 1
* ed25519 pubkey: 0xb52f5366eab0f5d95d0472a2fb4b2741a4bcfaafd3d563747bc93796577861a6
* public excess: 0x02ef03d2f597453151ea7a87aa32493bbd347fa26128007038db69c48ef0687acb
* public nonce: 0x028fc8de02ae4a3e2c1d82e6c007b7a970012a9b49498688451eca170d3af50b6f
* signature: 0xf6ff51be2540a5d3000027dc29073c04f4fcf57c39c5519534e8cb9d1b5718691069e36a28dab2d3b1b86b9ebbc963f8da1a2c28f495c87a7bbb0ee19b48520e

#### Bech32-encoded address
grin1qx6j75mxa2c0tk2aq3e2976tyaq6f0864lfa2cm500yn09jh0ps6vqh0q0f0t969x9g757584geyjwaax3l6ycfgqpcr3kmfcj80q6r6evpgljx7q2hy503vrkpwdsq8k75hqqf2ndy5np5gg50v59cd8t6skmlklagmuf2q5hfsqqp8ms5sw0qy7n702lpec4ge2d8geww3k4ccdygxncm29rdt95a3hp4eaw7fv0ud5x3v9r6ftjr60wasacvmfpfquglvavs

## Payment Proofs

### Building the proof

Calculate `proof_nonce` = `Hash(sender.public_nonce | receiver.address | tx.amount)`, where `receiver.address` is the decoded one-time address (`version | ed25519_key | public_excess | public_nonce | signature`)

The `total_nonce` for the transaction will then be `sender.nonce + receiver.nonce + proof_nonce`.

* `e = SHA256(M | sender.public_nonce + receiver.public_nonce + (proof_nonce * G) | sender.public_excess + receiver.public_excess)`
* The sender's partial signature will be `sender.partial_sig = sender.nonce + e * sender.excess`
* The receiver's partial signature will be `receiver.partial_sig = receiver.nonce + e * receiver.excess` 

The final aggregated kernel signature `(s, k*G)` can then be calculated as `(sender.partial_sig + receiver.partial_sig + proof_nonce, total_nonce * G)`

### Verifying the proof

The following 4 steps are necessary for the sender to prove payment to the receiver's address:

1. Provide the kernel and show that it was confirmed on-chain.
2. Prove knowledge of `sender.nonce`
3. Show that `sender.public_nonce + receiver.public_nonce + (proof_nonce * G)` = `total_nonce * G` (ie. the `k*G` in the kernel signature)
4. Provide the preimage to `proof_nonce` (`sender.public_nonce | receiver.address | tx.amount`)

## Slate Format

A slate is now only needed for passing information from the sender to receiver. It must include the following data:

* `tx.amount`
* `tx.offset`
* `tx.kernel_features`
* `tx.fee`
* `tx.inputs`
* `receiver.change_output`
* `receiver.public_excess` & `receiver.public_nonce` (to prevent the need for grinding)
* `sender.public_excess`, `sender.public_nonce`, & `sender.partial_sig`
* `time_to_live` (optional)

TODO: Define JSON & binary formats

## Backward Compatibility

Deciding whether to use the traditional method of sending or the new approach using one-time addresses should be straightforward for wallet developers. If provided a one-time address, use the new method. Otherwise, use v4 slates and follow the existing process.

TODO: Do we want to phase out traditional 4-step transaction building?

## Invoices

To send an invoice to a payer, the recipient provides the payer with a newly generated one-time address, an amount, and any additional fields we decide to support (memo, refund address, expiration, etc). The payer, upon approving payment of the invoice, simply follows the standard 2-step send process described in this RFC. There's no need for a special workflow.

## Security

### Nonce Reuse

Nonces should be randomly generated by the receiving wallet, rather than derived from seed, since reuse of a nonce could lead to loss of funds. Receiving wallets should keep track of all one-time addresses (excesses & nonces) they generate in a list (`address_list`), along with an indicator for whether they have been used yet.

Upon receipt of a partial transaction (slate), the receiver should obtain a lock on `address_list` and then check whether the address has been used before. If it has not, it's safe to finalize the transaction, but before broadcasting, the address should be marked as used in the `address_list`, and then the lock can be safely released.

### Play Attacks

TODO: Describe play attacks, and finish this section.

Any inputs sent from a wallet to a one-time address should be considered as sent. The sender should have the option to add additional inputs and/or increase the fee (equivalent of bitcoin's RBF).

Receiver should be unable to claim a problem with the transaction. If they do, it's trivial to show them the slate that was sent, and prove that it's a valid partial transaction containing everything necessary for the receiver to finalize. If they still wish to cancel, the sender must spend those inputs back to themselves.

# Drawbacks
[drawbacks]: #drawbacks

Addresses are significantly longer (272 characters).

Using one-time addresses increases the risk of "Play" attacks [2]. Recommendations for dealing with these have been discussed in the above "Play attacks" section.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

An alternative approach to simplifying transaction building is to support non-interactive transactions [3].

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* Can this be used to simplify sending to a multisig address?

# Future possibilities
[future-possibilities]: #future-possibilities

Eliminating finalize brings us one step closer to supporting receive-only wallets, and even hardware wallets. Offline wallets could pre-generate dozens, hundreds, or even thousands of address-nonce pairs, which can be given to a receive-only wallet which distributes them to potential senders, and collects & verifies any partial transactions provided to them. At any time in the future, those partial transactions can be provided to the offline wallet for signing, without needing to later contact the sender.

# Credits

Thanks to Kurt Coolman for coming up with an earlier iteration of the payment proofs design [5].

# References
[references]: #references

[1] https://forum.grin.mw/t/eliminating-finalize-step/7621
[2] https://forum.grin.mw/t/play-attacks-and-possible-mitigations/7527
[3] https://github.com/litecoin-project/lips/pull/13
[4] https://github.com/mimblewimble/grin-rfcs/blob/master/text/0015-slatepack.md#slatepackaddress
[5] https://forum.grin.mw/t/eliminating-finalize-step/7621/22