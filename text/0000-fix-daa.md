- Title: fix-daa
- Authors: [John Tromp](mailto:john.tromp@gmail.com)
- Start date: July 23, 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

Change Grin's DAA (Difficulty Adjustment Algorithm) from the current damped simple moving average (dsma) to a weighted-target exponential moving average (wtema). Optionally further restrict the future-time-limit window to 1 minute.

# Motivation
[motivation]: #motivation

The current DAA suffers from oscillatory behaviour, and also requires keeping the last 60 block on hand.
wtema is simpler to state, requires only the last 2 blocks, and fixes the oscillatory behaviour.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

Grin aims for a 1 minute average block time.
Thus, if the last block took exactly 1 minute to solve, then network difficulty remains the same in Grin's DAA.
If it took 2 minutes to solve, then network difficulty should be decreased.
But only by a small percentage. Halving network difficulty would make it way too erratic. 
Similarly, if the last block took less than a minute to solve, then network difficulty should be increased a little.
Grin's DDA uses a simple formula to express the factor by which to change network difficulty, in terms of the ratio of last block time to ideal block time.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

In wtema-M, the factor by which to lower network difficulty after every block
is 1 + (t/T - 1) / M, where T is the ideal block time of 60 seconds, t is the
last block time in seconds, and M is a mean life time.  Grin requires strictly
increasing timestamps in consecutive blocks, so t > 0.
In M blocks, the difficulty can thus be increased by at most (1-1/M)^-M ~ e.

# Drawbacks
[drawbacks]: #drawbacks

The one drawback I can think of is that using the difference of consecutive
timestamps makes the DAA more susceptible to timestamp manipulation. To quanify
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

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The simplest fix to the current DAA to reduce the problem of oscillations is to increase the damping factor.
The reason Grin went with a small damping factor was to make the DAA very responsive to changes in graphrate.
This responsiveness will suffer from increasing the damping factor. It makes sense to consider changing not just
a parameter in the current DAA, but the DAA altogether, as alternatives may offer better trade-offs between stability
and responsiveness. They may also achieve greater overall simplicity, both conecptually and in implementation.

# Prior art
[prior-art]: #prior-art

The current DAA in Bitcoin Cash suffers from the same oscillatory behaviour that Grin's DAA does.
Which is what led various BCH developers to do extensive research on possible alternatives [1] [2].
They identified two DAAs as being superior to others. These two, wtema, and asert, also happen to behave nearly identically.
While wtema is simpler in that it doesn't need exponentiation, it would need special safeguards to robustly deal with
the possibility of very negative solvetimes. Since Grin requires strictly increasing timestamps, it doesn't need any such
safeguards, making wtema the preferred choice for Grin.

Before adding a dependence on number of parent uncles, Ethereud uses a DAA that can be seen as a close approximation of wtema:
After every block, increase network difficulty by a factor of 1 - (t/T - 1) / M.
Comparing with wtema above, we see that instead of dividing by (1+x), they multiply by (1-x).
Even though Ethereum, like Grin, enforces positive solvetimes, they still need to guard against division by 0 in this form.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
While this RFC argues for adopting wtema as a new DAA, it doesn't want to argue for a specific value of the half life M.
In Grin tradition, an integral number of hours is preferable, to limit
arbitrariness. Either 2 hours, 3 hours, or 4 hours, all offer a resonable
balance of responsiveness and stability.
These 3 choices as well as Grin's current DAA with various damping factor choices are implemented at [3],
which allows for easy comparison across many scenarios.
At the same time, we need to decide if reducing the maximum allowed clock drift (FTL parameter) from 12 minutes
down to 1 makes sense, as this will reduce one form of timestamp manipulation.

# Future possibilities
[future-possibilities]: #future-possibilities

Although Grin graphrate has been relatively stable throughout its short history, as a GPU mineable coin availble at nicehash, it's always at risk of sudden temporary surges in graphrate, to which it should be able to respond quickly. The current DAA does that, but at a cost (of oscillatory behaviour).
In a future where Grin mining is dominated by Cuckatoo32 ASICs and where Grin is the C32 coin with the largest daily issuance, 
the graphrate should be naturally stable, with the need for responsiveness lessened.
We should however account for the possibility of other coins adopting Cuckatoo32 as well.
If, like most coins, they opt for a finite supply, then in initial years they may well have a daily issuance exceeding that of Grin. That would put us in a situation similar to Bitcoin Cash competing with Bitcoin for ASIC hashrate.
Having such a well vetted DAA will be a reassurance for years to come.

# References
[references]: #references

[1] https://www.yours.org/content/the-wtema-difficulty-adjustment-algorithm-855a3405606a

[2] https://read.cash/@jtoomim/bch-upgrade-proposal-use-asert-as-the-new-daa-1d875696

[3] https://github.com/tromp/difficulty
