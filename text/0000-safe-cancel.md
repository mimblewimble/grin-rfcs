
- Title: safe-cancel
- Authors: [John Tromp](mailto:john.tromp@gmail.com)
- Start date: Oct 10, 2019
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

## Summary
[summary]: #summary

Allow for safe cancellation of pending transactions, preventing future so-called play attacks.

## Motivation
[motivation]: #motivation

A wallet cannot simply cancel a pending transaction by forgetting about it and returning its inputs to the wallet balance.
Especially not when the other party is responsible for completing and broadcasting the transaction.
They may still do so at any time as long as the inputs are not spent differently.

## Community-level explanation
[community-level-explanation]: #community-level-explanation

Suppose Bob as the receiver sends Alice as the sender an invoice slatepack
(Receiver-Sender-Receiver or RSR flow) to which Alice responds, but then Bob
doesn't finalize it.  He might suggest a problem with the invoice, with his
wallet, or the exchange rate, or the (suddenly realized)
need to pay in a different currency. Whatever the case, he convinces Alice to
have the current transaction cancelled.  Alice is fine with that, but needs to
make sure that Bob doesn't later complete the transaction and steal Alice's
inputs.

Alice has two options; spend a transaction input back to herself (a self-spend), or
construct a new transaction that shares at least one input with the
old transaction. The former is cleaner, but requires separate fees, and possibly waiting
for confirmation.

The wallet provides a command for each of these options.

grin-wallet respend [OPTIONS]

will mark the pending transaction, specified either by ID or TxID UUID,
as requiring one of its inputs to be spent in the very next wallet spend.

grin-wallet unspend [OPTIONS]

will generate and broadcast a self spend of an input of 
the pending transaction, specified either by ID or TxID UUID,

The problem is not limited to RSR flow. In the more common SRS flow, once the
sender has signed and published the transaction, it can still fail to confirm.

It could be that one of the nodes in the Dandelion stem drops it, either by
accident or on purpose.  Alternatively, it could be explicitly rejected by the
mempool.
The receiver can arrange for that to happen by (slightly earlier) publishing
another transaction with an identical output.  When two transaction conflict by
sharing either an input or an output, the first to enter the mempool will block
the second from entering.

In case the sender's wallet notices that failure to confirm is due to a
conflicting transaction, it should alert the user that the receiver is being
malicious. 

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Drawbacks
[drawbacks]: #drawbacks

The user is required to understand the trade-offs between the two types of cancel.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The simpler alternative is to always do a self spend. Giving the user control
over whether to spend extra fees seems preferrable though.

## Prior art
[prior-art]: #prior-art

Coins such as Monero suggest the use of self spends to reduce linkability, but our motivation is quite different,
and with likely no aggregation in the Dandelion stem phase, self-spends in Grin do nothing to reduce linkability.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

## Future possibilities
[future-possibilities]: #future-possibilities

## References
[references]: #references

Include any references such as links to other documents or reference implementations.

- [reference 1](link)
- [reference 2](link)