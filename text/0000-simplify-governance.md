
- Title: `simplify-governance`
- Authors: [@paouky](mailto:ar.gis@protonmail.com), [@lehnberg](mailto:daniel.lehnberg@protonmail.com)
- Start date: `Sep 11, 2020`
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

## Summary
[summary]: #summary

This governance iteration replaces the previous process that was set out in [RFC#0002](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0002-grin-governance.md) with a simplified and better-defined version. Specifically, it:

- defines its remit to be around the repos, projects, and communities that are centered around the [/mimblewimble](https://github.com/mimblewimble) GitHub organization;
- reverts the core team to the original technocratic council structure, whose sole responsibility is managing the general fund; and
- explicitly enables individual contributors and teams to self-organize for other aspects of decision making.

## Motivation
[motivation]: #motivation

RFC#0002 was an iteration on governance in response to the sudden disappearance of Ignotus Peverell, the founder of Grin. Introduced together with the RFC process as defined in [RFC#0001](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0001-rfc-process.md), it served its purpose to formalize a structure for how the project would organize and make decisions. Over time however, it became clear that while the structure had its benefits, it came with its own drawbacks.

### Flaws of the previous structure

1. **All critical decision making became centralized around the core team.**
   - Consensus changes
   - Releases
   - Spending decisions
   - RFC approvals
   - Project leadership
1. **It established a negative feedback loop for contributors:**
   1. It created an “us” and “them” in the community; Those on the inside (part of the core team), and those on the outside (not part of the core team).
   2. This in turn led to the assumption that the core team were “responsible” for the project’s direction and success, which fostered complacency by non-core members.
   3. This in turn led to more work for the core team, which made it more tolling to be a member of it, reinforcing the notion of "us" and "them" in point 1 above.
1. **The core team structure would either suck up or turn away contributors.** Either: there were no new core team members added, which would be bad for contributor morale, and bad for the legitimacy of the structure;
   Or: those that contributed the most in the community would be added, which would lead to only core team members being the strong contributors. Which in turn would reinforce the “us” and “them” situation described above.
1. **Despite all the authority vested in the core team, there were no real checks and balances.**
   - Core team members had the right to stay for an indefinite time, there were no terms.
   - Only core members could add new core members or remove existing ones.
   - Community members could protest and object in case of conflict, but nothing technically would prevent the core team from going against the will of the community.
1. **Sub-teams had few non-core team members active.** Because of the general lack of non-core engagement, it ended up being core team members filling up most of the spots in the sub-teams, defeating their purpose.
1. **The remit and scope of governance was left undefined.** Was the process supposed to govern Grin the wider network? Or Grin the implementation under the [/mimblewimble](http://github.com/mimblewimble/) github organization? How does the grin forum and grin.mw website fit into this? There were a lot of ambiguities as to what exactly was being governed.

### Objectives of the new structure

1. **Establish a clear scope.** Make it explicitly defined what the new structure applies to.
1. **Keep things simple.** Less is more. One of Grin's design objectives is minimalism. This should be applied to its governance model, as much as anywhere else.
1. **Empower contributors.** Make it easy for new contributors to get involved, pick up tasks, and feel responsible for some part of the project.
1. **Reduce centralization and single point of failure risk.** Avoid having all key decisions go through a single structure.

## Community-level explanation
[community-level-explanation]: #community-level-explanation

The new structure can be broken down into three areas: Scope, Teams, and Participation.

### Scope

The explicit remit of this governance structure is defined as all the repos that are organized under the GitHub organization [/mimblewimble](https://github.com/mimblewimble) along with its existing and future communities and websites. This will be referred to as "the Project" below for clarity purposes, which include among other:

- [/grin](https://github.com/mimblewimble/grin), [/grin-wallet](https://github.com/mimblewimble/grin-wallet), and [/grin-miner](https://github.com/mimblewimble/grin-miner)
- [/grin-pm](https://github.com/mimblewimble/grin-pm), [/grin-rfcs](https://github.com/mimblewimble/grin-rfcs), and [/grin-security](https://github.com/mimblewimble/grin-security)
- [https://grin.mw](https://grin.mw)
- [https://forum.grin.mw](https://forum.grin.mw)
- keybase [@grincoin](https://keybase.io/team/grincoin) team
- [Mimblewimble mailing list](https://lists.launchpad.net/mimblewimble/)
- [Grin General Fund](https://grin.mw/fund)

### Teams

There is no hierarchical organization, and there is no "official" representative or group of representatives of the Project. Teams form and self-organize based on need. Teams are any group that carry out specific tasks or have specific responsibilities or have privileged access to tools or services related to the Project. They are responsible for their own processes and policies.

#### Technocratic Council

The Core Team as per RFC#0002 is removed. The existing members form the team that was known as the Technocratic Council prior to RFC#0002. The main purpose of the council is to manage the keys to the Grin General Fund and have final say in spending decisions. The technocratic council does not have a say in the matters of other teams.

#### Other teams

Examples of existing teams with various degree of organization that are seamlessly transitioned into their own autonomous structures include:

- node development team
- wallet dev team
- security team
- forum, keybase, and mailing list admins and moderators

### Participation

- Each team is responsible for how to add or remove team members. Where practical, open and unrestricted participation is encouraged.
- As much as possible of discussion and team organization should be made in public. Note-keeping is encouraged to make it easier for newcomers to get up to speed.
- As with any other aspect of the Project, contributors are required to adhere to the [Code of Conduct](https://grin.mw/policies/code_of_conduct).

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This section covers some of the more technical details of the process.

### Naming

Where there needs to be clarification between Grin (the implementation in [/mimblewimble/grin](https://github.com/mimblewimble/grin)), and Grin (the wider network consensus running any number of different implementations), it is suggested that [existing conventions](https://grinnode.live/stats) are followed and the former is referred to by the user agent name `MW/Grin` or `mw/grin`.

### RFC process

The RFC process as defined in RFC#0001 remains unchanged, and is how "significant changes" are proposed. Teams can assign RFCs to themselves if RFCs relate to their areas. Other community members are free to opine on proposals.

### Funding

The funding process remains as described in [RFC#0014: General Fund Guidelines](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0014-general-fund-guidelines.md), with the difference that it is the Technocratic Council who now has final say on spending decisions.

### Consensus-breaking changes

Network wide consensus rules are out of scope of this process. Significant changes to mw/grin are introduced via the existing RFC process.

### Disagreements / Dissent

- Teams are expected to resolve their internal disputes to the best of their abilities.
- Dysfunctional teams are replaced by forming new teams and contributors are encouraged to migrate over.
- Where there is disagreement between teams, other teams and community members can be asked to mediate.

## Drawbacks
[drawbacks]: #drawbacks

- **Lack of final decision-making authority.** With no clear decision-making process to fall back on in case of major disagreements, a community split, and therefore a network fork, becomes more likely.
- **Hierarchy is still present, but no longer as visible.** There are still some contributors with decision making authority, and some without. Before, this was codified explicitly. Now this is subtle and fluid.
- **Tasks may fall in between cracks easier.** Without a team with ultimate responsibility picking up slack, some things might simply not get done and progress may slow.
- **Lack of contributors may lead to the same people in charge everywhere.** Similar to the previous structure, if there are not enough new contributors joining, it ends up being the same people occupying all the different positions.

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

[A forum post](https://forum.grin.mw/t/dismantling-the-core-team-and-governance-structure/) on forum.grin.mw solicited ideas for alternatives to the previous structure. Some of those included:

- Establishing different teams being responsible for funds, code repos, and direction;
- Separating consensus protocol decisions from repository decisions;
- Spending the General Fund more aggressively in order to try out new directions and ideas;
- Simplifying or removing the RFC process altogether; and
- Creating a "House of Representatives" to balance out the powers of the Core team.

Additional options include:

- Doing nothing; and
- Attempting to establish a governance process that involves the entire network, current (and future?) implementations.

The rationale for pursuing the specific approach that is outlined in this RFC is because it:

- Establishes a scope for the process, which is contained and well defined;
- Simplifies the structure significantly compared to what was before;
- Has been tried in the past in the project, with some success;
- Can be implemented easily, and does not carry much migration risk;
- Can easily be evolved further by introducing new structures and processes if and when those are mandated; and
- Is not likely to leave the project significantly worse off than before.

## Prior art
[prior-art]: #prior-art

This replaces RFC#0002 with something closer to the governance process that was in place earlier in the project's life cycle.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- What is the governance process between different implementations?
- How can the funds and the spending decisions of the General Fund become more decentralized, removing the function of the Technocratic Council?
- How can commit rights to the Project repos be formalized better?

## Future possibilities
[future-possibilities]: #future-possibilities

- Should we stop accepting donations to addresses controlled by a single entity?

## References
[references]: #references

- [RFC#0002: grin-governance](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0002-grin-governance.md)
- [RFC#0001: rfc-process](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0001-rfc-process.md)
- /mimblewimble GitHub organization: [https://github.com/mimblewimble](https://github.com/mimblewimble)
- [https://forum.grin.mw/t/dismantling-the-core-team-and-governance-structure/](https://forum.grin.mw/t/dismantling-the-core-team-and-governance-structure/)
