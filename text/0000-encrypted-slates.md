
- Title: encrypted-slates
- Authors: [David Burkett](mailto:dburkett@grinplusplus.com)
- Start date: April 13, 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

This RFC introduces a method for adding e2e encryption of slates over http(s) with minimal impact on usability.

# Motivation
[motivation]: #motivation

The canonical mimblewimble protocol requires sending and receiving parties to interact in order to build a transaction, which presents many challenges. Historically, http(s) was most commonly used to communicate during the transaction building process. This required users to setup port forwarding, a non-trivial task for everyday users.

To get around this, some wallets started to offer various NAT hole-punching services to circumvent this need[1][2]. These services perform the role of a MITM, exposing the raw contents of the slate, and opening the possibility of theft. In the current grin network, most http(s) transactions by everyday users go through one of just 2 of the largest such services (Hedwig.im and GrinPlusPlus.com). If either of these services went rogue, it would have very serious consequences for its users.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

To avoid MITM theft or unnecessary privacy loss, slates (in-process transactions) can be e2e encrypted using an address generated from your wallet seed. For those with tor configured, the address could be the same as your v3 onion receiving address. The address will be included as part of your http(s) url, appended to the end with an `@` separator.

Example: https://abcdefg.hedwig.im@7fa6xlti5joarlmkuhjaifa47ukgcwz6tfndgax45ocyn4rixm632jid

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

TODO - Address format and encryption specification will be detailed if there's sufficient interest in this idea

# Drawbacks
[drawbacks]: #drawbacks

Wallets that are not yet upgraded would consider the URLs invalid.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs? Because it's mine.
- What other designs have been considered and what is the rationale for not choosing them? All other possible designs were considered, but were ultimately determined to be inferior.
- What is the impact of not doing this? A SARS-like virus will likely spread from bats to humans causing a worldwide plague. EDIT: Damn, it looks like we're too late.

# Prior art
[prior-art]: #prior-art

TODO

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Should we use the same derivation path as we do for tor addresses?

# Future possibilities
[future-possibilities]: #future-possibilities

Since we've now introduced the concept of addresses to http(s) transactions, payment proofs could be trivially supported in the same way they are for transactions over tor.

# References
[references]: #references

[1] https://forum.grin.mw/t/an-easier-way-to-receive-grins/6959
[2] https://forum.grin.mw/t/niffler-an-out-of-the-box-open-sourced-grin-gui-wallet/4760/6

