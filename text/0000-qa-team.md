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

The day to day impact is more resources to empower contributors to make successful contributions to Grin by performing quality assurance testing and review on their own work. The long term impact is more consistency in the level of quality work that makes it into the Grin project over time.

## Example

In this example a large PR is submitted that makes a nontrivial change to the Grin node.
* Before submitting the PR, the contributor:
  * reads and integrates the well-formed contribution documentation to assist and support their efforts
  * uses available testing frameworks to test their work
* Reviewers of the PR may use the documentation and tooling support provided by the QA team to perform another round of quality assurance testing and review on the submission
* Any necessary feedback is shared with the contributor to empower them to continue to make high quality contributions

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Membership

* Membership is bootstrapped by @lehnberg and @j01tz
* New membership approved with consensus of existing team members
* Members may be removed by self-resignation, consensus decision by QA team members or by core team

## Decision Making

* A consensus-seeking approach is used for decision making through dialogue and discussion, modeled after the general principle in the Grin governance RFC[1].
* In the event the team cannot reach consensus on a required action, the core team will determine the outcome

## Responsibilities

The responsibilities of the QA team _do_ include:
  * Establishing and providing the necessary support for contributors to have a clear path to success by empowering them to perform quality assurance testing and review on their own work
  * Clarifying, monitoring and enforcing the following of best QA practices by contributors

The responsibilities of the QA team _do not_ include:
  * Doing the quality assurance testing and review work themselves
  * Writing tests for code submissions and changes
  * Keeping testing frameworks updated based on codebase changes
  * Manually reviewing each submission for quality assurance
  * _Was the codebase changed? Then the tests must also be changed or extended for full coverage- this is not the responsibility of the QA team!_

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
[1] https://github.com/mimblewimble/grin-rfcs/blob/master/text/0002-grin-governance.md#general-principles
