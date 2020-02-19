- Title: qa-team
- Authors: [joltz](mailto:joltz@protonmail.com)
- Start date: Feb 12 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

* The QA team for Grin:
  * works to ensure that a high standard of quality for contributions is upheld across the project
  * develops tools and processes to help monitor and enforce the quality of work
  * exists to make sure that changes to the project are quality, not to produce the quality changes themselves
  * ensures that contributors are encouraged, supported and empowered to produce the best quality work to be integrated into Grin

# Motivation
[motivation]: #motivation

Over time, with many people contributing to many different areas of Grin, some things can get lost in the mix including quality. It could be beneficial to have a team supported by members, processes and tools to help provide a consistent level of quality contributions.

* Ensure that contributions to Grin are high quality
* Empower contributors to make high quality contributions to Grin
* Provide feedback from a QA perspective to micro and macro changes in Grin

# Community-level explanation
[community-level-explanation]: #community-level-explanation

The QA team observes micro and macro changes occurring over time with various contributions to Grin. They help review PRs, have opinions on both development and governance decisions and provide tooling frameworks for contributors to use and maintain to ensure a consistent high level of quality of work across the Grin project. The QA team provides support for contributors to encourage them to continue to submit quality work.

The day to day impact would be more attention for contributions and contributors. The long term impact would be more consistency in the level of quality work that makes it into the Grin project over time.

## Example

In this example a large PR is submitted that makes a nontrivial change to the Grin node.
* Before submitting the PR, the contributor reads and integrates the well-formed contribution documentation to assist and support their efforts
* Before the PR is accepted the contributor may use the Grin testing harness framework provided by the QA team.
* Members of the QA team may manually review to changes for quality at a micro level: does the code do a good job of implementing the feature, is it readable, are there any obvious security issues, does it follow our contribution guidelines for formatting etc.
* The QA may also review the change for quality at a macro level: are the changes in line with the other pieces of the project, does it add inconsistencies when interacting with other parts, does it advance the expected overall quality of the project etc.
* Any necessary feedback is shared with the contributor to empower them to continue to make high quality contributions

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Membership

* Membership is bootstrapped by @lehnberg
* Minimum three, maximum seven members
* New membership approved by majority decision of existing team members
* Members may be removed by self-resignation, majority decision of QA team members or by core team

## Decision Making

* Contentious decisions are resolved by majority decision
* In the event the team cannot reach a majority decision the core team will resolve the dispute

## Responsibilities

Roles and responsibilities of the QA team are listed here.
* Establish and make clear the standards of quality for contributions such that contributors feel supported by the process and have a clear path to success
* Review changes across mimblewimble organization repos for quality and provide feedback
* Provide micro and macro input regarding quality for development and governance changes
* Provide frameworks for test tooling in an effort of complete unit and integration coverage
* Empower contributors to make high quality contributions with documentation, tools and constructive feedback

## Community Contributions

The QA team is ultimately responsible for creating an environment that encourages and empowers contributors to make quality contributions to Grin. To this end, the QA team will strive to provide the resources necessary for contributors to make consistent, high quality contributions to Grin, including but not limited to: documentation, constructive feedback, tooling and clarity of expectations of quality.

# Drawbacks
[drawbacks]: #drawbacks

* Resources are already spread thin, don't want to assign responsibilities if people won't have time

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The alternative is a volunteer based approach to QA. This often leaves areas not properly assessed for quality that volunteers do not find interesting are already too busy making or reviewing other contributions. A volunteer based QA approach does not establish any accountability or oversight to changes in quality over time, nor does it provide active efforts to support new contributors.

# Prior art
[prior-art]: #prior-art

The QA team RFC is following the guidance of the establishment of teams in the governance RFC[0].

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* To what extent does the QA team develop tools?
* What is the priority of quality in a contentious change with competing factors like core opinion, community opinion and security?
* How can we best maintain a balance of strictness of quality and openness to all levels of contributions and contributors?

# Future possibilities
[future-possibilities]: #future-possibilities

* Full, robust testing support for all areas of Grin
* Improved processes to support contributors

# References
[references]: #references

[0] https://github.com/mimblewimble/grin-rfcs/blob/master/text/0002-grin-governance.md#teams
