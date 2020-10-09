
- Title: early-payment-proofs
- Authors: [John Tromp](mailto:john.tromp@gmail.com)
- Start date: Oct 7, 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [Edit if merged with link to tracking github issue]
---

# Summary
[summary]: #summary

Support generating and validating payment proofs for all transactions, including a timestamp and memo field.

# Motivation
[motivation]: #motivation

Payment proofs prevent a payment receiver from claiming they didn't receive payment.
Such fraud prevention is an essential ingredient to commercial adoption.

Former payment proofs used in Grin didn't apply to invcoice flow, at least not
without additional rounds of communication, which made invoices a difficult proposition.

They also lacked the ability to specify dating and purpose of payment.

This RFC changes the transaction building process where payers can require
payees to create a "proof" they've received a payment before the payer
finalizes and broadcasts the transaction.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

A payment receiver, in their earliest round of communication to a payment sender,
shall provide a signed statement of the following:
appearance of certain on-chain data satisfying certain conditions
shall constitute payment for a specified purpose.

This early provision of a payment proof commitment enables its use in all possible transaction building flows.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Slate changes

The slate will include the following payment proof related fields:

* `receiver_address` - An ed25519 public key for the receiver, typically the public key of the user's v3 onion address.
* `timestamp` - The time at which the receiver generates the payment proof
* `memo` - A string of limited size that may contain additional payment details
* `receiver_signature` - A signature that validates against the `receiver_address`
   over a payment message consisting of the following fields:
  - receiver public nonce
  - receiver public excess
  - `sender_address`
  - `timestamp`
  -  amount
  - `memo`

Note that although included in the signature, the `sender_address` need not be included in the slate,
as both sender and receiver necessarily  know it. Slates should only contain the absolute minimum of information
that needs to be relayed to the other party.

## Generating proofs

When the receiver generates their nonce and excess, they also generate the receiver signature.

## Witnessing a Payment Proof

A payment proof witness is a pair (s,i) satisfying s\*G = R + e\*X, where R is the receiver public nonce, X is the receiver public excess, and e is the hash challenge of the on-chain kernel with MMR index i.
A payment proof consists of a receiver signature together with a corresponding witness.

## Verifying Proofs

This `payment_proof` can be provided by the sender at any time to convince a payee that a payment was made to them.
The proof can be verified as follows:

1. Ensure that the i'th kernel has sufficiently many confirmations.
2. Verify that s\*G = R + e\*X, where e is the hash challenge of the i'th kernel.
3. Verify that the `receiver_signature` is valid.

## Wallet actions

### receiver excess generation

The `receiver_signature` is generated and added to the slate as part of the receive tx-building step.

### sender partial signing

Before signing, the sender verifies the receiver signature and checks the payment details.

# Drawbacks
[drawbacks]: #drawbacks

* Addition of timestamp and memo increase the size of tx slates.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

# Prior art
[prior-art]: #prior-art

* Wallet713 implements payment proofs for grinbox transactions, which our design adapts and builds on to work more seemlessly with onion addresses and with transaction building methods that don't inherently rely on addresses.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* What limit is imposed on the size of the memo field?

# Future possibilities
[future-possibilities]: #future-possibilities

# References
[references]: #references

* [Payment Proofs RFC](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0006-payment-proofs.md)
* Beam's payment proof model: https://github.com/BeamMW/beam/blob/c9beb0eae55fa6b7fb3084ebe9b5db2850cf83b9/wallet/wallet_db.cpp#L3231-L3236
