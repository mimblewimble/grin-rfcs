# GRIN-RFC-0002 Grin Governance

```
- Number: GRIN-RFC-0002
- Title: Grin Governance
- Status: Draft
- Authors: Daniel Lehnberg (daniel.lehnberg@protonmail.com)
           Michael Cordner (yeastplume@protonmail.com)
- Created : June 24th, 2019
```

# Summary
[summary]: #summary

Evolve Grin's governance:

- Introduce RFC (Request For Comment) process
- Introduce self-governing sub-teams that steward and guide work in each of their focus areas.
- Convert council into core team and define its responsibilities
- Recognize more contributors
- Achieve less centralization

# Motivation
[motivation]: #motivation

### Background

Grin's governance today consists of the Grin Technocratic Council, and the rest of the community. The council came to be as there was a need for some sub-set of community members to manage the multi-sig keys of the Grin General Fund, and it became a modest first pass at a governance structure. Over time, it has come to be that a lot of responsibilities and day to day tasks are handled by Grin council members. This puts a heavy workload on council members, but it also inhibits other members of the community from becoming more engaged, contributing more, and becoming recognized for it. Naturally this implies that either the council would have to grow large in order to be able to fit all the members of the community that deserve recognition for their contributions, or that we end up not rewarding active contributors properly.

As most decisions are taken in the bi-weekly meetings, it's come to be that decisions take time to materialize: It can take up to 14 days just to have the initial conversation and sometimes multiples of that to reach an agreement. In these meetings the entire active community ends up participating in every discussion in a synchronous fashion that often does not end up being productive.

We also struggle to answer more fundamental questions about Grin's governance, as in how decisions are made, who has authority to make these, and what the path is for a community member who would like to take on greater responsibilities.

This is a proposal to evolve our governance process.

### Objectives

- **Reward and recognize contributions better.** Offer different ways to become more engaged and give better recognition for work 
- **Empower more to contribute.** Encourage more community members to participate, facilitate more initiative.
- **Make Grin less centralized.** Rather than relying on a small group of people, share responsibilities and make the project more resilient against shocks.
- **Create a more transparent process.**

### Not objectives

- **Do not create fiefdoms.** Do not create emperors. Nobody is anybody else's boss.
- **Do not create bureaucracy for the sake of bureaucracy.** Do not impose death by a thousand papercuts or let forms and administration get in the way of making progress.
- **Do not discourage contributions.** Nobody can prevent someone else from doing work. Anyone can contribute in which ever way they find meaningful. You do not need to ask for permission.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

This proposal outlines a set of loose principles to guide the work we do. While some of these may already be in use, they might not have been articulated before. It converts the council into a core team, and outlines its responsibilities for the first time. In addition, it's proposed that additional teams are introduced alongside it, as well as an RFC process. An initial teams breakdown is suggested, and the proposal concludes with a path to adoption.

### General principles

- **Lead by example.** "Cypherpunks write code." We don't tell others what to do. We do what we can, and if we need to we ask for help. We suggest, but never command. We act as we want others to act. 
- **Not a democracy.** We are evaluated based on the work we do. It's not a popularity contest, and the majority is not always right. Community members are here by their free will, participation is optional.
- **Influence is measured by recent and not historical work.** We are grateful for and respect historical contributions, but they do not lead to lifelong positions of authority. Influence is earned by making contributions consistently over time, allowing new contributors to join the ranks of the old. 
- **Transparency.** Where possible, discussions and decisions are made in the open. We have nothing to hide, and we do not try to limit oversight unless there's a defensible reason to do so.
- **Keep things lightweight.** We strive to only put in place the minimal structure and organization that's needed, neither more nor less.   
- **Groups organize themselves.** Structures do not need to be imposed tops down, and we recognise that what works for one group will not necessarily work for the other. Teams self-organize and define their own workflows and processes as they see fit.  
- **Consensus-seeking decision making.** Voting creates winners and losers, and is polarizing. We recognize there are trade offs with everything and rarely any single right answer. This does not mean design by committee. We seek consensus through dialogue and discussion, but where there is a lack of consensus, we do not let it block us indefinitely. We're ready to make judgment calls to the best of our abilities. 
- **We speak for ourselves.** We can only speak in the name of ourselves, as contributors to the project. We do not write blog posts, articles, tweets, or give interviews speaking on behalf of the project as a whole. Grin itself has no single voice.   
- **There's no need to ask for permission.** We are not afraid to take decisions. We ask for feedback and opinions from others, but we do not need to ask for somebody's permission. If we believe it's in the best interest of  Grin, we act, and are accountable for our actions. 
- **Mistakes are tolerated.** As with any organization structure, mistakes happen. This is understood, and mistakes are accepted. We try to learn from them and improve. We assume we all act in good faith, until proven otherwise.

### Core team

The existing grin council is proposed to be renamed to core team.  The core team leads the wider Grin project as a whole. In particular:

* **Sets overall direction and vision.** Core values and philosophies. Steering towards use cases and targets.
* **Sets priorities and plans releases.** Maintains high level planning, roadmaps, focus areas, decides on pace and the release schedule.
* **Work on broader, cross-sectional issues.** What falls in-between sub-teams, taking a global view. 
* **Adds and removes sub-teams.** While proposals for new sub-teams can come from anywhere, the core team is responsible to ensure structures are productive and make sense.
* **Responsible for security.** Handles disclosures, vulnerabilities, audits, processes. 
* **Handles multi-sig keys and takes high level spending decisions.** Spending proposals can be made by anyone, and sub-teams can have their own own budgets to deal with as they please.

The core-team is self-selecting, and *may include* sub-team members. Over time, it's intended to be a broad representation of the wider community with diverse stakeholders.

### RFC process

An RFC is a Request For Comments document, outlining a proposed improvement or design change to an area of Grin. The exact specifics for the template is TBD. They are kept in their own dedicated repo and need to be accepted before a pull request is merged. Their purpose is to outline a standardized way of making proposals and allow the community to discuss and evaluate whether something is worth doing. Having an RFC accepted means that there's support "in theory" for the suggestion. It does not mean that a change becomes implemented automatically or in the exact way it is proposed, it is high-level design. The work still needs to be carried out. Accepted RFCs guide the broader planning work.

### Sub-teams

Sub teams are groups organized around specific areas or knowledge fields. They are responsible for these areas, but do not do all the work. Anyone can contribute anywhere, and do not need to hold a particular title to do so. 

Rather, sub-teams work on policies, processes, and workflows for their specific areas, as required. They are in charge of the RFC process in their specific field: They determine what requires an RFC in their area, they assign RFC shepherds that guide an RFC through its various stages and ensures the right stakeholders become aware of it and solicit their feedback. Ultimately, sub-teams decide whether an RFC in their area should be accepted or rejected. They are responsible to ensure that each RFC in their area has a tracked status, and that they progress towards an outcome.

Sub-teams self-organize. They should have a leader, often this leader may be part of the core team. They determine how members get added to the team. They should include area experts, and stakeholders. 

Sub-teams can be broken down into smaller working groups or sub-teams, permanent or temporary, as required and is seen necessary for them to be productive.

Each sub-team has a dedicated section on the forum, they should meet regularly, and keep some notes on what was covered and decided. Decisions do not need to happen in meetings, and could for example be handled asynchronously or in the RFC process.  

### The teams

In addition to Core team, the following teams are proposed to be created initially.

* **Node development.** Core Grin technology, changes and optimizations to the  node and anything consensus related, research and discussion of new technologies, proof of work.

* **Wallet development.** wallet technology, wallet API, wallet-related research.

* **Infrastructure.** Technical documentation, non-technical documentation, QA, testing, toolchain, developer productivity, guides, how-tos.

* **Ecosystem.** 3rd party developers interaction (wallets, pools, exchanges and others), integration and technical assistance, growing the grin ecosystem, stakeholder collaboration.

* **Community.** Onboarding of new community members, website, chat channels, conferences, events, meetups.

* **Fundraising.** Sponsorships, donations, activities to increase project funds.

* **Moderation.** Code of conduct, handles violations, across all areas of the project.  To avoid biases and conflicts of interest, this sub-team _does not_ contain any member of the core team.

### Adoption path

1. Publish Draft proposal in this respository
2. Discuss and refine as required
3. If approved;
   - Core Team: 
      1. Delineate roles, responsibilites and powers of core team?
      2. Other open questions see [unresolved-questions](#unresolved-questions)
   - Sub-teams:
      1. Open questions as above
      2. Appoint team leads from council/core where available
      3. Let sub-teams without leaders self-form once there is someone ready to step up
   - RFC process:
      1. Create a repository (this one)
      2. Propose, discuss and approve template & process document as grinrfc-0001 and grinrfc-0001

# Drawbacks
[drawbacks]: #drawbacks

TBD

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

TBD

# Prior art
[prior-art]: #prior-art

TBD

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* Is this change to the structure actually going to help? Will we just end up with the same general arrangement of people as currently exists under a slightly different guise?
* Is the core and subteam structure too loose and informal? Do we need to resolve questions such as:
    * Who chooses the core team?
    * Assuming the core team has a leader, what are the roles and responsibilities/veto/tie breaking powers etc?
    * How long is their tenure? How do you get rid of them?
    * Same questions for subteams
    * The difference between the sub-teams and core teams. Where is the authority delineated?
    * What happens during disagreements between the core team and subteams? Or internal disagreements?
* Should we consider a core team and a council (see Python Steering Council Model below)

# Future possibilities
[future-possibilities]: #future-possibilities

TBD

# References
[references]: #references

**Rust's governance process**
- https://rust-lang.github.io/rfcs/1068-rust-governance.html
- https://predictablynoisy.com/rust-governance
- http://mgattozzi.com/oss-governance-and-sustainablility-i/

**Node.js governance**
- https://medium.com/the-node-js-collection/healthy-open-source-967fa8be7951

**Python Steering Council Model**
- https://www.python.org/dev/peps/pep-8016/

**Swarmwise, by Rick Falkvinge**
- https://falkvinge.net/files/2013/04/Swarmwise-2013-by-Rick-Falkvinge-v1.1-2013Sep01.pdf
