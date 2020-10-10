
- Title: `Add Payjoin support`
- Authors: Phyro
- Start date: `Oct 10, 2020`
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

## Summary
[summary]: #summary

Support for PayJoin transactions which help break the [common input ownership heuristic](https://en.bitcoin.it/wiki/Common-input-ownership_heuristic).

## Motivation
[motivation]: #motivation

Current Grin transactions are very similar to Bitcoin's in the sense that we have a single party sending the coins - the sender. There are usually two receivers, the sender receives
a change output and the receiver receives the amount they have been sent. The side effect of this is that we can know that all the inputs must have the same owner which allows us to link them to the same identity.
One way to mitigate this was described in [Pay2EP](https://blockstream.com/2018/08/08/en-improving-privacy-using-pay-to-endpoint/) post where it suggests to introduce optional interactivity to Bitcoin transactions
which allows the receiver to contribute inputs to the transaction. This allows the transaction to have more than 1 sender because the receiver can also spend their inputs. Such a transaction is called a payjoin,
and having such transactions on chain would improve the overall privacy. Grin already has interactive transactions, so we just need to provide the wallet flow to construct them.

### PayJoin transactions
PayJoin transactions are one possible way to break the common-input-ownership heuristic. They achieve this by having the receiver contribute some inputs to the transaction.
It might seem like payjoin transactions only benefit the sender while hurting the privacy of the receiver, but the receiver also wants to spend outputs in an input set with a mixed ownership.
Each transaction is an opportunity for the receiver to spend outputs on the input side that will have mixed owners. Doing so comes at a cost of showing the contributed receiver's
inputs (usually just one) to the sender, but also at a benefit of having multiple owners of inputs which should in most cases be in the interest of both the sender and the receiver.
Important thing to note here is that if there are very few or no payjoins happening on the network, this same receiver could in a future transaction spend multiple inputs together and
show their spent inputs not only to the transacting party, but to everyone else as well, which seems like a worse privacy tradeoff than doing payjoins. Spending multiple inputs together
is not uncommon, so we want most of the transactions be payjoins to help us obfuscate our own inputs when we spend them. A positive side effect of doing a payjoin as a receiver is that
we consolidate some of the outputs to create an output that holds more coins, so we are less likely to come in a situation where we need to spend multiple inputs together.
Having the majority of the transactions be payjoins is also beneficial for the whole network because it creates probabilistic input ownership which makes backwards chain analysis much harder to do since we can't tell whether an input belongs to the sender of the receiver.

Currently, PayJoin transactions are cheaper than regular transactions because they have more inputs. A 2-2 PayJoin transaction does not create a new output as opposed to a regular 1-2 transaction,
which means that it can be thought of as an replacement of the value of the current output with a new value which does not increase the UTXO set size.

PayJoin transactions also allow for the possibility of the receiver paying their share of fees. Whether users would find this useful is not clear yet.

## Community-level explanation
[community-level-explanation]: #community-level-explanation

A wallet user should be able to choose whether they will create a payjoin or a regular transaction. Payjoin transactions should only be done through the invoice transaction flow to prevent
possible UTXO snooping attacks where the sender could initiate a transaction (or many), get the information about an input from the receiver and never finalize it.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

TODO
- how does that look for the user exactly? Who decides who will share the address first?

### Exchanges

Exchanges could support withdrawals as payjoin transactions. This means that they would share the address with the receiver which would initiate an invoice flow for the receive transaction.
It does mean that the exchange would need to associate an address with the amount and user identity on the exchange because the receiver could pick any amount when starting the invoice flow.

Since the withdrawal transaction might not arrive on the chain (e.g. the user disconnecting) in which case the exchange would need to cancel the transaction, it means that the user has to pay for the fees
of the transaction _regardless_ whether the transaction lands on the chain or not. In both cases, when it lands and when the exchange needs to cancel the transaction, it should not be the exchange covering the fees.

For similar reason, exchanges could avoid doing non-payjoin transactions to avoid the need for cancellation for deposit transactions.

## Drawbacks
[drawbacks]: #drawbacks

TODO

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

## Prior art
[prior-art]: #prior-art

I don't think any wallet implementation supported payjoins due to the lack of payment proofs in the invoice flow. This has been solved with [early payment proofs](https://github.com/tromp/grin-rfcs/blob/early-payment-proofs/text/0000-early-payment-proofs.md).

## Unresolved questions
[unresolved-questions]: #unresolved-questions
- Does this mean that exchanges should have two options? regular/payjoin withdraw? or perhaps just a payjoin where the user finalizes and not the exchange?
- What does the user finalizing the tx mean for the exchange with regards to canceling a tx?
- Is an invoice flow _always_ a payjoin transaction?

## Future possibilities
[future-possibilities]: #future-possibilities

TODO

## References
[references]: #references

- [Pay2EP Blockstream](https://blockstream.com/2018/08/08/en-improving-privacy-using-pay-to-endpoint/)
- [Pay To Endpoint](https://medium.com/@nopara73/pay-to-endpoint-56eb05d3cac6)
