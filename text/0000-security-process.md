
- Title: security-process
- Authors: [John Woeltz](mailto:joltz@protonmail.com)
- Start date : July 18, 2019
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

This RFC improves the security processes for Grin by adopting a community
standard for a public pre-commitment scheme for vulnerability sharing agreements.
The standard proposed for adoption describes the norms for ethical disclosure
behavior and provides a public pre-commitment scheme for the Grin community to
clarify security process actions and expectations [2].

# Motivation
[motivation]: #motivation

- Improves transparency around Grin security and disclosure processes
- Decreases reaction time for mitigating security incidents across the Grin ecosystem
- Increases preparedness in the event of an extreme vulnerability
- Improves overall ecosystem resiliency
- Provides clarity around ethical behavior and expected deviations
- Makes possible the mapping of the vulnerability disclosure surface

# Community-level explanation
[community-level-explanation]: #community-level-explanation

This proposal adopts a vulnerability disclosure standard for Grin that provides
more guidance and clears up some unanswered questions from Grin's current
security process [4]. The community should think of this improvement as an extension
to the existing security process with the following changes:

- Better define ethical behavior (and deviations) for vulnerability disclosure
- Better define the processes and expectations for receiving and sending disclosures
- Provide a framework for bi-lateral disclosure agreements with other projects/implementations
- A public pre-commitment to the above

This all results in a more robust security process for Grin that is transparent
from the beginning to the community, vulnerability researchers, the core team
and other neighboring projects that share threads in the same security blanket.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

_The standard that this RFC proposes to adopt for Grin is located at https://github.com/RD-Crypto-Spec/Responsible-Disclosure.
Below we will describe how the standard would be adopted by Grin's Security Process
in SECURITY.md, based on the example used by Zcash [3]._

The 'Code Reviews and Audits' and 'Chain Splits' sections will be removed as
they are out of scope for SECURITY.md.

Details for 'Code Reviews and Audits' can be kept track of on project forums or
in the 'grin-pm' repo or relevant sub-team repo. The status and resource availability
for code reviews and audits will change over time and will likely be changing
too often to be stable enough to be included in a formal security policy.

Likewise 'Chain Splits' is not defined well enough and would probably be better
tracked and managed outside of the formal security policy, or with a sub-team,
at least until the language and tooling are better defined and stable.

_The sections below will replace the 'Responsible Disclosure', 'Vulnerability
Handling' and 'Recognition and Bug Bounties' sections of SECURITY.md following
a statement of intention to adhere the standard as well as its location._

### Receiving Disclosures

>Grin is committed to working with researchers who submit security vulnerability
notifications to us to resolve those issues on an appropriate timeline and perform
a coordinated release, giving credit to the reporter if they would like.

Emails and PGP keys for Grin's security contacts will be listed here as in SECURITY.md

### Sending Disclosures

>In the case where we become aware of security issues affecting other projects
that has never affected Grin, our intention is to inform those projects of
security issues on a best effort basis.

>In the case where we fix a security issue in Grin that also affects the following
neighboring projects, our intention is to engage in responsible disclosures with
them as described in https://github.com/RD-Crypto-Spec/Responsible-Disclosure,
subject to the deviations described in the 'Deviations from the Standard' section.

### Bilateral Responsible Disclosure Agreements

Here we would list any agreements we have with neighboring projects to share
vulnerability information in accordance with the standard itself and the
'Deviations from the Standard' section in the adoption of the standard.
Agreements would be made by the core team or security sub-team based on capacity
to engage and relevance/impact on Grin's ecosystem and to an extent the greater
cryptocurrency ecosystem.

### Deviations from the Standard

>Grin is a technology that provides strong privacy. Due to the nature of Grin's
zero-knowledge commitments and rangeproofs, if a counterfeiting bug results it
could be be exploited without a way to identify when or which data was corrupted,
making a rollback or other fork-based attempted fix ineffective.

>The standard describes reporters of vulnerabilities including full details of an
issue, in order to reproduce it. This is necessary for instance in the case of
an external researcher both demonstrating and proving that there really is a
security issue, and that security issue really has the impact that they say it
has - allowing the development team to accurately prioritize and resolve the issue.

>In the case of a counterfeiting or privacy-breaking bug, however, we might
decide not to include those details with our reports to partners ahead of
coordinated release, so long as we are sure that they are vulnerable.

# Drawbacks
[drawbacks]: #drawbacks

This proposal could add complexity and overhead that a donation-supported decentralized
project may not be able to uphold in good faith. Adopting and pre-committing
to a standard and policy is not effective if it cannot be upheld through actions.
This is amplified with bilateral agreements that extend through chains of vendors
sharing code, many forks and multiple implementations with many operating publicly.

For a project with extremely limited resources (relative to large companies)
and no source of steady funding, it may not be possible to support the time and
resources required to follow through on such a commitment.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

#### Why is this design the best in the space of possible designs?

This seems to be the most transparent and least centralized option that still
provides the ability to efficiently handle cases of severe vulnerabilities that
may be unique to Grin, while keeping with the minimalist philosophy.

Relying on a single security@ disclosure email address provides a single point
of failure. Additionally, not having clear pre-commitments to sharing agreements
and expected ethical deviations can cause community contention at best and
failure to successfully mitigate a vulnerability before it is exploited at worst.

#### What other designs have been considered and what is the rationale for not choosing them?

A less centralized model would be to attempt to guide this process with on-chain
governance or an on-chain bug bounty program. Both of those not only challenge
Grin's minimalist philosophy but are also extremely complicated to execute on.
These do not seem to be viable options for Grin at this time.

Ideally formal verification could be used to handle many of these security
challenges but there is a huge gap between what formal verification can actually
do and the complex blockchain related things we are doing today that we want
formally verified. This may be more viable in the future, but it would require
significant time and resource investment. It would not be responsible to rely
on formal verification for a project like Grin today.

#### What is the impact of not doing this?

If we do not adopt a standard as proposed here, the community is left with
unresolved problems:

- Who do we tell if we receive a vulnerability?
- What details are we obligated to share?
    - Exploit code?
    - What if it is an inflation bug?
- Do we attempt to collect a bug bounty?

The community is exposed to risk by not having clear answers to these questions
_before_ it is presented with a critical security vulnerability. Even if the
vulnerability has technically been fixed, there may be severe fallout in the
community if it perceives the situation wasn't handled ethically by the parties
involved in the disclosure. There is also risk of not implementing a fix in time
across the community because there were internal disputes about who to tell.

Without a public pre-commitment to expected actions taken during a vulnerability
disclosure, Grin runs the risk of alienating the community in pursuit of
security, or even worse, not implementing a critical fix before any user
experiences privacy or value loss.

# Prior art
[prior-art]: #prior-art

The traditional disclosure model (RF Policy/security@) [0] handles interaction between
researcher and vendor but does not quite fit for our use case of potentially multiple
vendors up/down stream. There have been multi-party models proposed with single
researcher and multiple vendors that have not been successful, nor do they
sufficiently account for the challenges faced in public decentralized projects.

Neither model fits with chains of vendors sharing code, protocols with multiple
implementations and forks, all operating in public. We need a model that is
better suited to handle cases of consensus critical vulnerabilities that require
upgrades across many implementations, forks and projects both in the Grin and
greater cryptocurrency ecosystems.

Bitcoin has received some criticism over handling of disclosures, as it never
had a robust, well-defined pre-committed standard to follow. We want to learn
from those lessons to avoid putting more people in the position to repeat the
same mistakes.

Zcash, a privacy based cryptocurrency, experienced a critical vulnerability
that inspired the creation of the standard this RFC proposes to adopt [1]. Due
to the similar nature of the Zcash and Grin ecosystems, this prior art is highly
relevant to the security process for Grin.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How can we further reduce centralization risk for Grin's security process?

- How can we responsibly handle vulnerabilities in academic papers?

- What role will formal verification play in Grin's future?

- What form, if any, will the Bug Bounty process take?

# Future possibilities
[future-possibilities]: #future-possibilities

- Security sub-team
    - If a security sub-team is adopted in the future it would likely maintain these processes
- On-chain governance/bug bounties
- Bug bounty program
- Formal verification

# References
[references]: #references

Include any references such as links to other documents or reference implementations.

[0] https://dl.packetstormsecurity.net/papers/general/rfpolicy-2.0.txt

[1] https://www.youtube.com/watch?v=h7W1u1K2VjQ

[2] https://github.com/RD-Crypto-Spec/Responsible-Disclosure

[3] https://github.com/zcash/zcash/blob/master/responsible_disclosure.md

[4] https://github.com/mimblewimble/grin/blob/09cf6de1d143ffbe007478372dc573213e06804d/SECURITY.md
