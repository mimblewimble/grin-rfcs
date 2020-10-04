- Title: fix-fees
- Authors: [John Tromp](mailto:john.tromp@gmail.com)
- Start date: August 23, 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

## Summary
[summary]: #summary

Change Grin's minimum relay fees to be weight proportional and make output creation cost about a Grin-cent.
Restrict fees to 40 bits, using 4 of the freed up 24 bits to specify the desired tx priority,
and leaving the remaining 20 bits for future use.
NOTE: this is a hard-forking change.

## Motivation
[motivation]: #motivation

The current fee requirements suffer from being somewhat arbitrary, and not miner incentive compatible.
They are not even linear; to avoid a negative minimum fee, they are rounded up to zero.
As a consequence, the minimum fees for the aggregate of two
transactions is not necessarily equal to the sum of the individual ones.
Worse, a miner has no incentive to include a transaction that pays 0 fees, while they take resources to relay.
The current (and foreseeable) low price of Grin makes spamming the UTXO set rather cheaper than desired.
Fee overpaying, for higher priority to be included in full blocks. fails when aggregated with minimal fee transactions.

## Community-level explanation
[community-level-explanation]: #community-level-explanation

For clarity, let's adopt define the following definitions. Let the `feerate' of
a transaction be the fees paid by the transaction divided by the transaction
weight with regards to block inclusion (`Transaction::weight_as_block`).
Let the `minfee' of a transaction be the amount of fees required for relay and mempool inclusion.
A transaction paying at least `minfee' in fees is said to be relayable.

Grin constrains the size of blocks by a maximum block weight, which is a linear
combination of the number of inputs, outputs, and kernels.  When blocks fill
up, miners are incentivized to pick transactions in decreasing order of feerate.
Ideally then, higher feerate transactions are more relayable than lower feerate ones.
Having a lower feerate transaction be relayable while a higher feerate transaction is not
is very much undesirable and a recipe for problems in a congested network.

The only way to avoid this mismatch between relay and block inclusion incentives
is to make the minimum relay fee be proportional to the block weight of the
transaction. This leaves only the constant of proportionality to be decided.
We can calibrate the fee system by stipulating that creating one output should
cost at least one Grin-cent (formerly 0.4 Grin-cent).

We want minfee to be linear to make splitting and combining of fees well-behaved.
For instance, two parties in a payjoin may each want pay their own input and output fees,
while splitting the kernel fee, without overpaying for the whole transaction.

Only the leat significant 40 bits will be used to specify a fee, while
some of the remaining bits will specify a minimum fee overpayment factor,
preventing aggregation with lesser overpaying transactions, and earlier inclusion into full blocks.

The largest possible 40-bit fee is 2^40 - 1 nanogrin, or approximately 1099.5 grin,
which should be more than enough to guarantee inclusion in the next available block.
Technically speaking, transactions can pay higher fees by having multiple kernels,
but we still want to avoid that kind of friction.
As a side effect, this limits the damage of fat fingering a manually entered fee amount.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The minimum relay fee of a transaction shall be proportional to `Transaction::weight_as_block`,
which uses weights of `BLOCK_INPUT_WEIGHT` = 1, `BLOCK_OUTPUT_WEIGHT` = 21, and `BLOCK_KERNEL_WEIGHT` = 3,
which correspond to the nearest multiple of 32 bytes that it takes to serialize.
Formerly, we used Transaction::weight,
which uses arbitrary weights of -1, 4, and 1 respectively, but also non-linearly rounds negative results up to 0
(as happens when the number of inputs exceeds the number of kernels plus 4 times the number of outputs).

The `Transaction::weight_as_block` shall be multiplied by a base fee.
This will not be hardcoded, but configurable in grin-server.toml,
using the already present `accept_fee_base` parameter.
There is no reason to use different fees for relay and mempool acceptance.
Its value shall default to `GRIN_BASE` / 100 / 20 = 500000, which makes each output
incur just over 1 Grin-cent in fees.

The 64-bit fee field in kernel types Plain and HeightLocked shall be renamed to `fee_bits' and be composed of bitfields
{ `future_use': 20, `fee_shift': 4, fee: 40 }. All former uses of the fee will use `fee_bits' & `FEE_MASK',
where `FEE_MASK' = 0xffffffffff.

A nonzero fee shift places an additional restriction on transaction relay and mempool inclusion.
Namely, a transaction containining a kernel with `fee_shift' = s must pay a total fee
of at least 2^s times the minfee (the minfee shifted left by `fee_shift').

Aggregation of two relayable transactions should only happen when the result remains relayable.
By linearity, this is always the case for two transactions with identical fee shift.
Transactions with differing fee shift can be aggregated only if either one pays more
than required by their fee shift.

For instance, suppose two transactions have the same minfee. If one pays 3 times the minfee with a fee shift of 1,
while the other pays exactly minfee with a zero fee shift,
then both can be aggregated into one that pays twice the joint minfee (which is double the old one).

While a transaction can choose an arbitrarily high feerate to incentivize early inclusion into full blocks,
the fee shift provides for 16 different levels of protection against feerate dilution (through transaction aggregation).

The new tx relay rules and new fee computation in wallets shall take effect at
the HF4 block height of 1048320 (but see below about alternatives for 3rd party wallets).

The 20 bits of `future_use bits' provide a possible soft-forking mechanism.
We can imagine a future soft-fork further constraining the validity of kernels,
or the relayability of transactions containing them, depending on the value of some of these bits.

## Drawbacks
[drawbacks]: #drawbacks

Leaving 20 bits of `future_use' increases the amount of zero cost zero friction
spam that can be added to the chain.  However, we already have a similar amount
of spammable bits in the lock height of HeightLocked kernels, where any lock
height under the current height of around a million has no effect.
So this appears to be an acceptable increase.

At the time of writing, only one other perceived drawback could be identified, which is what motivated the former fee rules.
A negative weight on inputs was supposed to encourage spending many outputs, and lead to a reduction in UTXO size.
But this aligned poorly with miner incentives, as for instance they have no economic reason to include zero fee transactions.
A positive weight on inputs is no serious drawback, as long as creation of outputs is more strongly discouraged.
Most wallets, especially those relying on payjoins, do about equal amounts of output creation and spending,
the difference being just the wallet's UTXO set. 
Mining pools are the major exception. While they obviously don't incur any fees in coinbase creation, they need not
spend fees on spending them either, as they can mine these payout transactions directly.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

There are no good alternative to economically significant fees. Any blockchain lacking them is an open invitation to abuse.
For chains with a maximum blocksize, fees are also necessary to allow prioritization of transactions.

There is a small window prior to HF4 where transactions constructed using the former lower won't be finalized before HF4 and will thus fail to be relayed. Third party wallets are free to switch fee computation some arbitrary time before HF4 to minimize this risk.

Instead of a fee shift, one could specify a fee factor.
So then a transaction containing a kernel with the `fee_factor' bitfield having value f would require total fees
of at least f+1 times minfee (preserving old behaviour for a 0 bitfield).
A reasonable overpayment range would then require 8 bits instead of the 4 used for fee
shift, and would create 256 levels of priority. It does seem
wasteful though, to allow a distinction between, say, overpaying by 128 times,
or by 129 times.
Worse still, that many priority levels will severely reduce the potential for aggregation when blocks fill up.
To maximize aggregation opportunities then, forcing priority levels to be a factor of 2 apart seems quite reasonable.

We can avoid a consensus change by keeping the fee at the full 64 bits,
instead using its least significant 4 bits to specify a fee shift.
This only makes sense if we never plan to make any use of the top 24 bits, which would always be forced to 0
unless someone pays a monstrous fee, most likely by accident.
It would also lead to balances that are no longer a multiple of a milligrin, making for slightly worse UX.

## Prior art
Several chains have suffered spam attacks. In early days, bitcoin was swamped with feeless wagers on Satoshi Dice [1].
At least those served some purpose.

Nano was under a deluge of meaningless 0-value back and forth transfers for weeks,
that added dozens of GB of redundant bloat to its chain data [2]. Although nano requires client PoW as a substitute for fees,
these attacks showed that the PoW was poorly calibrated and not an effective deterrant.


## Unresolved questions
[unresolved-questions]: #unresolved-questions

## Future possibilities
[future-possibilities]: #future-possibilities

While the input and kernel weights are pretty fixed, the output weight is subject to improvements in range proof technology.
If Grin implements BP+, the weight should go down from 21 to 18. That would make outputs incur slightly less than a Grin-cent in fees,
which is not worth bothering with. If range proofs were to halve in size though, then we might want to double the base fee to compensate.

If Grin ever becomes worth many dollars, then a lowering of fees is desirable.
This can then be achieved by getting the majority of running nodes to reconfigure their base fee to a lower value,
after which wallets can have their fee computation adjusted.
In the converse case, where Grin becomes worth only a few cents, then an increase in fees might be needed to avoid spam.
Both cases will be much easier to deal with if they coincide with a hard fork, but those may not be on the horizon.

## References
[references]: #references

[1] [Satoshi Dice](https://en.bitcoin.it/wiki/Satoshi_Dice)

[2] [Nano github](https://github.com/nanocurrency/nano-node/issues/1883)
