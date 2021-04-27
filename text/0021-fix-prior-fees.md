- Title: fix-prior-fees
- Authors: [John Tromp](mailto:john.tromp@gmail.com)
- Start date: January 21, 2021
- RFC PR: [mimblewimble/grin-rfcs#0021](https://github.com/mimblewimble/grin-rfcs/pull/77)
- Tracking issue: https://github.com/mimblewimble/grin/issues/3635

---

## Summary
[summary]: #summary

Carry the restriction of fees, to 40 bits since HF4, back to all history.
Do not change `headerversion` despite the hard-forking nature.

## Motivation
[motivation]: #motivation

This makes the `FeeFields` methods `fee()` and `fee_shift()` height independent,
as well as several other functions which end up calling them.
This results in nontrivial code and consensus model simplification,
with no downside, which is always a good thing.

## Community-level explanation
[community-level-explanation]: #community-level-explanation

The largest fee to appear on mainnet prior to Hark Fork 4 is 2.404 Grin. This
means that extending the current 40-bit fee restriction to the pre-HF4 history
preserves validity.  While this is technically a hard-forking change, we could
treat it as an optional update.
An attacker could in theory split the network by creating an ultra-deep reorg
that includes a pre-HF4 fee exceeding 40 bits. However, the split damage would
utterly pale in comparison to having a many months deep reorg in the first
place.  So in practice this change does not justify a header version bump and
mandatory upgrade.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

When computing the fee paid by a transaction, take all the fee fields in transaction kernels, and  mask out all but the least significant 40 bits.
This masking used to be conditional on the header version of a argument height being at least 5 [1].

## Drawbacks
[drawbacks]: #drawbacks

None!

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

There is no compelling reason for leaving a height dependency in the fee method.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

How many months should we wait after HF4 before rolling out this code simplification?
I propose that 3 months is more than enough, making this RFC acceptable from mid April 2021 onwards.

## Future possibilities
[future-possibilities]: #future-possibilities

## References
[references]: #references

[1] [transaction.rs](https://github.com/mimblewimble/grin/blob/acba73bf40242f963d8ea1e7128dfdfde6fb8853/core/src/core/transaction.rs#L181-L188)
