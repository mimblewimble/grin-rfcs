
- Title: early-payment-proofs
- Authors: [John Tromp](mailto:john.tromp@gmail.com)
- Start date: Oct 7, 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [Edit if merged with link to tracking github issue]
---

## Summary
[summary]: #summary

Support generating and validating payment proofs for all transactions, including a timestamp and memo field.

## Motivation
[motivation]: #motivation

Payment proofs prevent a payment receiver from claiming they didn't receive payment.
Such dispute/fraud prevention is an essential ingredient to commercial adoption.

Former payment proofs used in Grin didn't apply to invoice flow, at least not
without additional rounds of communication, which made invoices a difficult proposition.

They also lacked the ability to specify the time and purpose of payment.

This RFC changes the transaction building process so that
payees commit to a proof of payment promise before the payer signs for the transaction.

## Community-level explanation
[community-level-explanation]: #community-level-explanation

A payment receiver, in their earliest round of communication to a payment sender,
shall provide a signature that makes the following promise:
appearance of certain on-chain data satisfying certain conditions
shall prove payment for a specified purpose.

This early provision of a payment proof promise enables its use in all possible transaction building flows.

To avoid confusion, the invoice flow shall be called receiver-sender-receiver (RSR) flow and
the "regular" flow shall be called sender-receiver-sender (SRS) flow.

The first byte of signed data will be used as payment proof type or version (similar to how we encode kernel features).

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Payment Proof is a Witnessed Promise

Since the receiver signature is a promise that payment will be considered proven by 
certain on-chain data satisfying certain conditions, we shall call such data a witness,
and let a payment proof consist of a promise paired with a witness.

### Slate changes

To accomodate the various proof types, the slate will include the following related fields:

* `receiver_address` - An ed25519 public key for the receiver, typically the public key of the user's v3 onion address.
* `timestamp` - The time at which the receiver generates the payment promise
* `memo` - A string of size at most 1024 bytes that contains payment details,
  such as a description of the good(s) or service(s) being bought.
  `memohash` denotes the 32 byte blake2b hash of this field.
* `promise_signature` - A signature that validates against the `receiver_address`
   over a promise message consisting of 1 byte of proof type, followed by type specific data

Note that although the sender address is commonly among the signed data, it need not be included in an slate that's encrypted for the sender, as both sender and receiver necessarily know it. Slates should only contain the absolute minimum of information that needs to be relayed to the other party.

### Proof type Legacy

These follow the original Payment Proofs RFC in the references.
The signature is over
  - proof type `0x00`
  - `amount`
  - `kernel_commitment`
  - `sender_address`

The witness is an on-chain kernel with the given commitment.
The proof type is actually the most significant byte of the amount field,
which is necessarily 0 in the foreseeable future.
The amount field is accordingly limited to 7 bytes.

### Proof type Invoice

This will be the type for regular invoices, where receiver specifies the time, amount and purpose of payment.  The signature is over
  - proof type `0x01`
  - `amount`
  - `receiver_public_nonce`
  - `receiver_public_excess`
  - `sender_address`
  - `timestamp`
  - `memohash`

The receiver will sign this data either in the first round of RSR flow, or the second round of SRS flow. In the latter case, the sender can use the slate fields `amount` and `memo` to set suggested values for the receiver to use. The `timestamp` should correspond to the time of signature generation.

For consistency with the old proof type, the amount is again limited to 7 bytes.
The witness is a triple (s,i,C) where i is the MMR index of an on-chain kernel K with commitment C, satisfying s\*G = R + e\*X, where R is the `receiver_public_nonce`, X is the `receiver_public_excess`, and e is the hash challenge of kernel K.
The reason for including the kernel index is that nodes don't maintain an index of all kernels, and looking for the index of a potentially very old kernel is rather expensive,
and proof verification should not be a DoS vector.
The reason for including the kernel commitment is so that the prover can recompute the index when necessitated by chain reorgs.

The specific steps in RSR flow are

Receiver fills in memo field and computes memohash. Receiver computes ed25519 signature over the message
  0x01 | amount | `receiver_public_nonce` | `receiver_public_excess` | `sender_address` | `timestamp` | `memohash`
and sets this as slate field `promise_signature`.

Upon receiving the slate, the Sender composes the same message from the various
slate fields and check the signature against the Receiver public ed25519 key.
The Sender inspects all payment details including the memo and presumably
agrees to proceed with payment. They save the partial signature sS of the
kernel they sign along with the kernel commitment and promise signature, and
send back the slate to the reciever for countersigning.

When the Sender wants to prove payment, they find the kernel with commitment C
on chain and compute sR = s - sS to recover the receiver partial signature sR
that will satisfy the payment proof condition.

Sender can convince any 3rd party of the payment to Receiver by providing both
the promise signature (including the memo and all message fields) and the witness.

### Proof type SenderNonce

This will be the type for indeterminate invoices, where sender commits by nonce to the time, amount and purpose of payment.
This roughly follows the payment proofs in David Burkett's Eliminate Finalize Step RFC (see References).

The signature is over
  - proof type `0x02`
  - 7 zero bytes
  - `receiver_public_nonce`
  - `receiver_public_excess`
  - `sender_address`

The receiver will sign this data in the first round of RSR flow, leaving the
sender to commit to remaining payment details in their following step.
There is no need for this proof type in SRS flow, as the simpler Invoice type suffices.

The witness is a quintuple (s,i,C,Rs',m) where i is the MMR index of an on-chain kernel K with commitment C, satisfying s\*G = R + e\*X, where R is the `receiver_public_nonce`, X is the `receiver_public_excess`, and e is the hash challenge of kernel K.
Additionally, the sender nonce Rs, computed as the difference between kernel nonce and receiver public nonce,
must be of the form Rs' + H(Rs' | m) \* G, where message m contains the promise fields
  - proof type `0x02`
  - `amount`
  - `timestamp`
  - `memohash`

This is similar to the Sign-to-Contract notion discussed in the last reference.

### Wallet actions

#### receiver excess generation

The `receiver_signature` is generated and added to the slate as part of the receive tx-building step.

#### sender partial signing

Before signing, the sender verifies the receiver signature and checks the payment details.

## Drawbacks
[drawbacks]: #drawbacks

* Addition of timestamp and memo increase the size of tx slates.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

In the SenderNonce proof type, instead of requiring the sender nonce to
commit to message m, we can require the kernel nonce to do so, which avoids
the need for a proof verifier to subtract the receiver public nonce from the
kernel nonce.

## Prior art
[prior-art]: #prior-art

* Wallet713 implements payment proofs for grinbox transactions, which our design adapts and builds on to work more seemlessly with onion addresses and with transaction building methods that don't inherently rely on addresses.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

## Future possibilities
[future-possibilities]: #future-possibilities

## References
[references]: #references

* [Payment Proofs RFC](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0006-payment-proofs.md)
* Beam's payment proof model: https://github.com/BeamMW/beam/blob/c9beb0eae55fa6b7fb3084ebe9b5db2850cf83b9/wallet/wallet_db.cpp#L3231-L3236
* [Eliminate Finalize Step RFC](https://github.com/DavidBurkett/grin-rfcs/blob/eliminate_finalize/text/0000-eliminate-finalize.md)
* [Pay-to-contract and Sign-to-contract](https://www.reddit.com/r/Bitcoin/comments/d3lffo/technical_paytocontract_and_signtocontract/)