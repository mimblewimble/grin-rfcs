
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
and having such transactions on chain would break the common input ownership heuristic and thus improve privacy. Grin already has interactive transactions, so we just need to provide the wallet flow to allow receivers
adding inputs.

#### PayJoin transactions
PayJoin transactions are one possible way to break the common-input-ownership heuristic. They achieve this by having the receiver contribute some inputs to the transaction.
It might seem like payjoin transactions only benefit the sender while hurting the privacy of the receiver, but the receiver also wants to spend outputs in an input set with a mixed ownership.
Each transaction is an opportunity for the receiver to spend outputs on the input side that will have mixed owners. Doing so comes at a cost of showing the contributed receiver's
inputs (usually just one) to the sender, but also at a benefit of having multiple owners of inputs which should in most cases be in the interest of both the sender and the receiver.
An important thing to note here is that if there are very few or no payjoins happening on the network, this same receiver could in a future transaction spend multiple inputs together and
show their spent inputs not only to the transacting party, but to everyone else as well, which seems like a worse privacy tradeoff than doing payjoins. Spending multiple inputs together
is not uncommon, so we want most of the transactions be payjoins to help us obfuscate our own inputs when we spend them. A positive side effect of doing a payjoin as a receiver is that
we consolidate some of the outputs to create an output that holds more coins, so we are less likely to come in a situation where we need to spend multiple inputs together.
Having the majority of the transactions be payjoins is also beneficial for the whole network because it creates probabilistic input ownership which makes backwards chain analysis much
harder to do since we can't tell whether an input belongs to the sender of the receiver.

Currently, PayJoin transactions are cheaper than regular transactions because they have more inputs. A 2-2 PayJoin transaction does not create a new output as opposed to a regular 1-2 transaction,
which means that it can be thought of as an replacement of the value of the current output with a new value which does not increase the UTXO set size.

PayJoin transactions also allow for the possibility of the receiver paying their share of fees. Whether users would find this useful is not clear yet.

## Community-level explanation
[community-level-explanation]: #community-level-explanation

A user should be able to choose whether they will create a payjoin or a regular transaction. Payjoin transactions should preferably be done through the invoice transaction flow to prevent
possible UTXO snooping attacks where the sender could initiate a transaction (or many) to gather the information about the receiver's outputs without the intent of actually accepting and broadcasting the transaction.
That said, it is possible to create a payjoin with a sender initiated flow, but it should only be done with a party that is trusted.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

A wallet configuration includes a new configuration key:

```yaml
srs_payjoin = True   # Use payoins for the sender-initiated flow
rsr_payjoin = True   # Use payoins for the receiver-initiated flow (invoice)
```

Both invoice and sender-initiated workflows use payjoin transactions as the default behaviour. The configuration defaults can be changed by the user to avoid doing payjoin in a specific flow.
A user can also manually opt-in/out of a payjoin by specifying a flag `--no-payjoin` to the command:

`grin-wallet invoice --no-payjoin -d grin1dhvv9mvarqwl6fderuxp3qgl6qpphvc9p4u24347ec0mvvg6342q4w6x5r 60`

or, if the configuration defaults to False for the current flow, they can opt-in to a payjoin by specifying a flag `--payjoin` to the command:

`grin-wallet receive --payjoin -d grin1dhvv9mvarqwl6fderuxp3qgl6qpphvc9p4u24347ec0mvvg6342q4w6x5r 60`


If no flag is given, the transaction is built based on the defaults for the current transaction building flow.
There should be no change apart from the wallet adding additional input from the receiver, the transaction flow should stay the same. Which inputs to pick is left as a decision to the wallet implementation.
In case the wallet does not have an available output to add as an input, it would create a non-payjoin transaction.

### Exchanges

Exchanges could use the flow that allows them to finalize the transaction in order to avoid having issues with locked outputs due awaiting finalization and possible canceling of transactions.

## Drawbacks
[drawbacks]: #drawbacks

Creation of a payjoin transaction through a sender-initiated workflow could be used as a UTXO snooping attack. In order to protect from this, we should only use payjoins in a sender-initiated flow with
parties that can be trusted not to do that.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

A possible alternative would be to not default to payjoin transactions.

## Prior art
[prior-art]: #prior-art

I don't think any wallet implementation supports payjoins due to the lack of payment proofs available for the invoice flow. This could be solved with [early payment proofs](https://github.com/tromp/grin-rfcs/blob/early-payment-proofs/text/0000-early-payment-proofs.md).

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should we default to no payjoins for sender-initiated flow?
- How does this impact privacy?

## Future possibilities
[future-possibilities]: #future-possibilities

Perhaps having a walllet configuration for payjoin use would make it easier for those that want to partially use payjoins or never use them.

## References
[references]: #references

- [Pay2EP Blockstream](https://blockstream.com/2018/08/08/en-improving-privacy-using-pay-to-endpoint/)
- [Pay To Endpoint](https://medium.com/@nopara73/pay-to-endpoint-56eb05d3cac6)
