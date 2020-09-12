- Title: onesided
- Authors: [John Tromp](mailto:john.tromp@gmail.com)
- Start date: September 7, 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

Allow for non-interactive transactions in cases where payment proofs are not needed.

# Motivation
[motivation]: #motivation

The current 2 round interaction required for building a transaction is somewhat cumbersome when requiring manual intervention.
In particular, the initiator has to wait an arbitrary time between sending their first and receiving the second message.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

Interaction can be minimized by splitting a transaction into two parts: a sending transaction and a receiving transaction,
similar to transactions in nano currency [1].
The sending transaction produces a 1-of-2 output whose blinding factor is a secret shared between sender and receiver.
The shared secret is constructed in a Diffie-Hellman key exchange, using known secp256k1 public keys.
The receiving transaction spends the 1-of-2 output into a 1-of-1 output under the sole control of the receiver.
The payment is not considered complete until the receiving transaction confirms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

[TODO]

# Drawbacks
[drawbacks]: #drawbacks

Transactions require double the fees and kernels. Doesn't allow for payjoins.

The major drawback though is that payment proofs are not possible, 
as a 3rd party could never distinguish between the sender spending the 1-of-2 and the receiver spending the 1-of-2.
Therefore, these one sided payments are only useful where payment proofs are redundant.
This includes tipping, donations, and any case where a recveiver fully trusts the sender,
like children receiving pocket money from their parents.

Constructing the shared secret requires the sender to pick a random secret key
whose corresponding public key must be passed on-chain to the receiver. This is unlikely
to fit in the limited space available in Bulletproofs, while creating an extra output field
is highly undesirable.

These drawbacks make this proposal unacceptable. It may still serve a documentary purpose on its way to rejection.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Alternative proposals [2] [3] [4] avoid the transaction splitting, but require a departure from pure Mimblewimble
while complicating the consensus model and possibly weakening the security model.

[6] argues for maintaining the simplicity and elegance of pure MW.

By using malleable signatures that don't commit to the public key (the public excess in kernels),
a 1 round interaction protocol may be possible within the pure MW consensus model [5].

# Prior art
[prior-art]: #prior-art

[TODO]

# Unresolved questions
[unresolved-questions]: #unresolved-questions


# Future possibilities
[future-possibilities]: #future-possibilities

[TODO]

# References
[references]: #references

[1] [Nano docs](https://docs.nano.org/integration-guides/key-management/#creating-transactions)

[2] [DavidBurkett gist](https://gist.github.com/DavidBurkett/32e33835b03f9101666690b7d6185203)

[3] [eprint](https://eprint.iacr.org/2020/1064.pdf)

[4] [Grin forum](https://forum.grin.mw/t/a-draft-design-of-mimblewimble-on-nervos-ckb)

[5] [Grin forum](https://forum.grin.mw/t/integrated-payment-proofs-and-round-minimization)

[6] [Grin forum](https://forum.grin.mw/t/pep-talk-for-one-sided-transactions/7361/8)
