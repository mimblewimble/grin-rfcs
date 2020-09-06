- Title: fix-fees
- Authors: [John Tromp](mailto:john.tromp@gmail.com)
- Start date: August 23, 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

Change Grin's minimum relay fees to be weight proportional and make output creation cost about a Grin-cent.

# Motivation
[motivation]: #motivation

The current fee requirements suffer from being somewhat arbitrary, and not miner incentive compatible.
They are not even linear; the fees required for the aggregate of two transactions is not necessarily equal to the sum
of required fees.
The current (and foreseeable) low price of Grin makes spamming the UTXO set rather cheaper than desired.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

Grin constrains the size of blocks by a maximum block weight, which is a linear combination of the number of inputs, outputs, and kernels.
When blocks fill up, miners are incentivized to pick transactions in decreasing order of fees per weight.
Ideally then, a transaction that is more preferred by miners, is also more likely to be relayed.
If a transaction A doesn't satisfy the minimum relay fees, while another transaction B does, but miners would rather include A,
then A's publisher will have to transmit it directly to miners, which is undesirable.
We therefore let the minimum relay fee be proportional to the weight of the transaction, minimizing arbitrariness.
We can calibrate the fee system by stipulating that creating one output should cost at least one Grin-cent (formerly 0.4 Grin-cent).
Linearity is another desirable property, which for instance allows the two parties in a payjoin to each pay their own input and output fees,
while splitting the kernel fee.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The minimum relay fee of a transaction shall be proportional to Transaction::weight\_as\_block,
which uses weights of BLOCK\_INPUT\_WEIGHT = 1, BLOCK\_OUTPUT\_WEIGHT = 21, and BLOCK\_KERNEL\_WEIGHT = 3,
which correspond to the nearest multiple of 32 bytes that it takes to serialize.
Formerly, we used Transaction::weight\_as\_block,
which uses arbitrary weights of -1, 4, and 1 respectively, but also non-linearly rounds negative results up to 0
(as happens when the number of inputs exceeds the number of kernels plus 4 times the number of outputs).
The Transaction::weight\_as\_block shall be multiplied by a base fee.
This will not be hardcoded, but configurable in grin-server.log.
The already present accept\_fee\_base parameter appears suitable for this, as
there as no reason to use different fees for relay and mempool acceptance. Its
value shall default to GRIN\_BASE / 100 / 20 = 500000, which make each output
incur just over 1 Grin-cent in fees.

# Drawbacks
[drawbacks]: #drawbacks

I think there is only a perceived drawback, which is what led to the former fee rules.`
A negative weight on inputs is supposed to encourage spending many outputs, and lead to a reduction in UTXO size.
A positive weight on inputs is not really a drawback however, as long as creation of outputs is more strongly discouraged.
Most wallets, especially those relying on payjoins, do about equal amounts of output creation and spending,
the difference being just the wallet's UTXO set. 
Mining pools are the major exception. While they obviously don't incur any fees in coinbase creation, they need not
spend fees on spending them either, as they can mine these payout transactions directly.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

There are no good alternative to economically significant fees. Any blockchain lacking them is an open invitation to abuse.
For chains with a maximum blocksize, fees are also necessary to allow prioritization of transactions.


# Prior art
Several chains have suffered spam attacks. In early days, bitcoin was swamped with feeless wagers on Satoshi Dice [1].
At least those served some purpose.

Nano was under a deluge of meaningless 0-value back and forth transfers for weeks,
that added dozens of GB of redundant bloat to its chain data [2]. Although nano requires client PoW as a substitute for fees,
these attacks showed that the PoW was poorly calibrated and not an effective deterrant.


# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities
[future-possibilities]: #future-possibilities

While the input and kernel weights are pretty fixed, the output weight is subject to improvements in range proof technology.
If Grin implements BP+, the weight should go down from 21 to 18. That would make outputs incur slightly less than a Grin-cent in fees,
which is not worth bothering with. If range proofs were to halve in size though, then we might want to double the base fee to compensate.

In Grin ever becomes worth many dollars, then a lowering of fees is desirable.
This can then be achieved by getting the majority of running nodes to reconfigure their base fee to a lower value,
after which wallets can have their fee computation adjusted.
In the converse case, where Grin becomes worth only a few cents, then an increase in fees might be needed to avoid spam.
Both cases will be much easier to deal with if they coincide with a hard fork, but those may not be on the horizon.


# References
[references]: #references

[1] https://en.bitcoin.it/wiki/Satoshi\_Dice

[2] https://github.com/nanocurrency/nano-node/issues/1883
