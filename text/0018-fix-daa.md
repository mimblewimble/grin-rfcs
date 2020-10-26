- Title: fix-daa
- Authors: [John Tromp](mailto:john.tromp@gmail.com)
- Start date: July 23, 2020
- RFC PR: [mimblewimble/grin-rfcs#61](https://github.com/mimblewimble/grin-rfcs/pull/61) 
- Tracking issue: [mimblewimble/grin#3473](https://github.com/mimblewimble/grin/issues/3473)

---

## Summary
[summary]: #summary

Change Grin's DAA (Difficulty Adjustment Algorithm) from the current damped simple moving average (dsma) to a weighted-target exponential moving average (wtema). Also, restrict the future-time-limit (FTL) window to 5 minutes.

## Motivation
[motivation]: #motivation

The current DAA suffers from oscillatory behaviour, and also requires keeping the last 60 block on hand.
wtema is simpler to state, requires only the last 2 blocks, and fixes the oscillatory behaviour.
The current FTL window of 12 minutes appears rather arbitrary. It matches the 2-hour window used in bitcoin
when expressed in their respective blocktimes, but we prefer a more natural amount in terms of clock time.


## Community-level explanation
[community-level-explanation]: #community-level-explanation

Grin aims for a 1 minute average block time.
Thus, if the last block took exactly 1 minute to solve, then network difficulty
remains the same in Grin's DAA.  If it took 2 minutes to solve, then network
difficulty should be decreased.
But only by a small percentage. Halving network difficulty would make it way
too erratic.  Similarly, if the last block took less than a minute to solve,
then network difficulty should be increased a little.
Grin's DAA uses a simple formula to express the factor by which to change
network difficulty, in terms of the ratio of last block time to ideal block
time.

A future-time-limit of 5 minutes naturally corresponds to the hour markers on a clock.
The trade-off for FTL is between impact of timestamp manipulation and risk of divergence between local
and actual clock time. For properly configured systems (e.g. correct time-zone and daylight-savings-time settings)
we expect drifts of several minutes to be quite noticeable, and likely to be corrected by operators.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

In wtema-M, the factor by which to lower network difficulty after every block
is 1 + (t/T - 1) / M, where T is the ideal block time of 60 seconds, t is the
last block time in seconds, and M is a mean life time.  Grin requires strictly
increasing timestamps in consecutive blocks, so t > 0.
In M blocks, the difficulty can thus be increased by at most (1-1/M)^-M ~ e.

The maximum allowed clock drift (FTL parameter) is to be 5 minutes; blocks whose timestamp exceeds
the local time by more than 5 minutes are ignored and not relayed.
This parameter is not hardcoded but to be specified in grin-server.toml

## Drawbacks
[drawbacks]: #drawbacks

At the time of writing, the only apparent drawback is that using the difference of consecutive
timestamps makes the DAA more susceptible to timestamp manipulation. To quantify
the effect of this, let's say that miners maximally misrepresent m typical
consecutive block times as one m-minute block + m-1 instant blocks. Instead of
keeping network difficulty constant, this would divide it by (1+(m-1)/M) * (1-1/M)^m.
For m much smaller than M, and M > 100, this is only a tiny increase in difficulty.
If m is as large as k\*M+1 then this is (approximately) dividing by (1+k)/e^k, i.e. multiplying by e^k/(1+k).
This is not something that miners should be interested in doing,
as they generally want to maximize rewards by minimizing difficulty,
so this kind of timestamp manipulation seems unlikely.
On the other hand, they have a small incentive to misrepresent block times as
being more regular (closer to T) than they really are, since for m blocks
taking a total time of m\*T, setting timestamps exactly 1 minute apart minimizes the new difficulty.
For instance, reporting a 30-second block and a 90-second block as two 1-minute blocks, changes the adjustment factor
from (1-1/2M)(1+1/2M) = 1-1/4M^2 to 1, which for M > 100 is pretty negligible, indistinguishable from noise
among realistic graphrate fluctuations.

The risk of NTP (Network Time Protocol) based local clocks diverging by more than 5 minutes is somewhat
larger than with a 12 minute threshold.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The simplest fix to the current DAA to reduce the problem of oscillations is to increase the damping factor.
The reason Grin went with a small damping factor was to make the DAA very responsive to changes in graphrate.
This responsiveness will suffer from increasing the damping factor. It makes sense to consider changing not just
a parameter in the current DAA, but the DAA altogether, as alternatives may offer better trade-offs between stability
and responsiveness. They may also achieve greater overall simplicity, both conceptually and in implementation.

An FTL window of 1 minute, matching the blocktime, was considered as being the least arbitrary,
and also leaving little room for timestamp manipulation. However, some worries remained that NTP is not robust enough to
practically guarantee sub-minute drift even for properly configured systems.


## Prior art
[prior-art]: #prior-art

The current DAA in Bitcoin Cash suffers from the same oscillatory behaviour that Grin's DAA does.
Which is what led various BCH developers to do extensive research on possible alternatives [1] [2].
They identified two DAAs as being superior to others. These two, wtema, and asert, also happen to behave nearly identically.
While wtema is simpler in that it doesn't need exponentiation, it would need special safeguards to robustly deal with
the possibility of very negative solvetimes. Since Grin requires strictly increasing timestamps, it doesn't need any such
safeguards, making wtema the preferred choice for Grin.

Before adding a dependence on number of parent uncles, Ethereum uses a DAA that can be seen as a close approximation of wtema:
After every block, increase network difficulty by a factor of 1 - (t/T - 1) / M.
Comparing with wtema above, we see that instead of dividing by (1+x), they multiply by (1-x).
Even though Ethereum, like Grin, enforces positive solvetimes, they still need to guard against division by 0 in this form.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

While this RFC argues for adopting wtema as a new DAA, it doesn't want to argue for a specific value of the half life M.
In Grin tradition, an integral number of hours is preferable, to limit
arbitrariness. Either 2 hours, 3 hours, or 4 hours, all offer a resonable
balance of responsiveness and stability.
These 3 choices as well as Grin's current DAA with various damping factor choices are implemented at [3],
which allows for easy comparison across many scenarios.

## Future possibilities
[future-possibilities]: #future-possibilities

Although Grin graphrate has been relatively stable throughout its short
history, as a GPU mineable coin available at nicehash, it's always at risk of
sudden temporary surges in graphrate, to which it should be able to respond
quickly. The current DAA does that, but at a cost (of oscillatory behaviour).
In a future where Grin mining is dominated by Cuckatoo32 ASICs and where Grin
is the C32 coin with the largest daily issuance, 
the graphrate should be naturally stable, with the need for responsiveness lessened.
We should however account for the possibility of other coins adopting Cuckatoo32 as well.
If, like most coins, they opt for a finite supply, then in initial years they
may well have a daily issuance exceeding that of Grin. That would put us in the
same situation as Bitcoin Cash competing with Bitcoin for ASIC hashrate.
Having such a well vetted DAA will be a reassurance for years to come.

Future improvements in network clock synchronization might make a reduction of FTL down to one minute viable.

## References
[references]: #references

[1] [archive.org](https://web.archive.org/web/20181120082228if_/https://www.yours.org/content/the-wtema-difficulty-adjustment-algorithm-855a3405606a/)

[2] [read.cash](https://read.cash/@jtoomim/bch-upgrade-proposal-use-asert-as-the-new-daa-1d875696)

[3] (tromp github)https://github.com/tromp/difficulty
