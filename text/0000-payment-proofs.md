- Title: payment-proofs
- Authors: [David Burkett](mailto:davidburkett38@gmail.com)

---

# Summary
[summary]: #summary

Support generating and validating payment proofs for sender-initiated (ie. non-invoice) transactions. 

# Motivation
[motivation]: #motivation

Bitcoin and other cryptocurrencies with transparent protocol-level addressing and immutable, unpruneable blockchains can prove sender, receiver, and amounts of payments simply by pointing to the transaction in the blockchain.
Grin's privacy and scalability means users no longer have this ability. This prevents some merchants from accepting Grin due to the high possibility of payment disputes that are unresolvable in the same way they are for transparent coins.

This RFC proposes a change to the transaction building process where payers can require payees to create a "proof" they've received a payment before the payer finalizes and broadcasts the transaction.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

From an end-user perspective, payers can require payees to prove receipt of funds as part of the transacting process.
Payers can then use these "proofs", along with information from the blockchain, to resolve payment disputes and prove funds were sent to the correct payee, and the transaction was confirmed on the blockchain.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Common workflow

## Slate changes

## Generating proofs

## Verifying proofs

## Wallet actions
### init-send

### receive

### finalize

# Drawbacks
[drawbacks]: #drawbacks

* Drawback 1
* Drawback 2

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* Can this be adapted to work for invoices?
* Question 2

# Future possibilities
[future-possibilities]: #future-possibilities

None

# References
[references]: #references

None

