# GRIN-RFC-0000 Full Wallet Lifecycle Support in Wallet API

```
- Number: GRIN-RFC-0000
- Title: Full Wallet Lifecycle Support in Wallet API
- Status: Draft
- Authors: yeastplume (yeastplume@protonmail.com)
- Created : June 26th, 2019
```

# Summary
[summary]: #summary

Increase the scope of the Grin Wallet's Owner API to support all wallet lifecycle functions. Wallet creation, seed management, (TBD)

# Motivation
[motivation]: #motivation

Grin Wallet's APIs currently provides functions for transacting and querying the contents of the wallet. However, several pieces of functionality
around wallet creation and seed/password management are not included within the API. This means that any consumers of the API will expect their users
to initialize the wallet manually before the APIs can be used.

The Wallet APIs are intended to be the foundation upon which community-created wallets should be built, and the job of a wallet creator is made far
more difficult by the absence of wallet creation and seed management functions within the API. Ideally, it should be the case that a wallet can
be instantiated and managed solely via the Owner API.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

Explain the proposal as if it were already included in the Grin ecosystem and you were teaching it to another Grin community member. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Grin community members should *think* about the feature, and how it should impact the way they use Grin. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Grin community members and new Grin community members.

For implementation-oriented RFCs (e.g. for wallet), this section should focus on how wallet contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Wallet Backend Structure and Multiple Wallets

The Wallet API should be able to support multiple named Wallets. Currently, the default wallet data directory assumes a single wallet instance, and
is structured as follows:

```
~/.grin/main/
   grin-wallet.log
   grin-wallet.toml
   wallet_data/
     wallet.seed
     db/
     saved_txs/
```

`wallet_data` contains the seed and database for a single instance of the wallet, with the path to the wallet's data directory configured in
`grin_wallet.toml`.

The structure of the wallet's data files will be changed to allow multiple wallet instances, with each named wallet contained in a directory
named according to name of the wallet provided by the end user. Current single-instance wallet data will be placed into a directory called
`default`, and the API and all wallet software will assume the default directory is to be used if no name is provided.

The wallet data directory structure will become (for example):

```
~/.grin/main/
   grin-wallet.log
   grin-wallet.toml
   wallet_data/
     default/
        wallet.seed
        db/
        saved_txs/
     my_wallet_1/
        wallet.seed
        ...
     my_wallet_2/
        wallet.seed
        ...
```

If a data directory is not provided in a wallet API call, it will be assumed to be operating on the `default` wallet

### Data migration

The target version of the wallet will contain a function to migrate existing wallets from the current data structure to the new,
essentially moving `wallet_data/wallet.seed`, `wallet_data/db/` and `wallet_data/saved_txs/` into `wallet_data/default/wallet.seed` etc.
(Should this be performed within the API for consistency?)

## Wallet Initialization

Currently, wallet data does not exist until the user runs `grin-wallet init`. The `init` command creates `grin-wallet.toml`,
in the `~/.grin/main` directory (or `~/.grin/floonet`, or the current directory via the `-h` flag), prompts the user for a password,
creates a seed file, stores the resulting data files in the directory specified in `grin-wallet.toml` (`~/.grin/main/wallet_data` by default)
and initialises the lmdb database.

OwnerAPI::create_config(Option<DataDirectory>) -> Outputs a `grin-wallet.toml` file into the given location, defaulting to `~/.grin/main/wallet_data`
OwnerAPI::create_wallet(Option<name>, password) -> Creates and initializes a new wallet in the directory specified by `name`, `default` if None
OwnerAPI::get_mnemonic(name) -> Returns mnemonic from given wallet

It should be possible to run `grin-wallet owner_api` or invoke the API directly from a linked binary without having instantiated a wallet.


(How is grin-wallet.toml generated?)

## Additional API Functions

Owner::SetWalletDirectory -> Set the top-level system wallet directory (`~/.grin/main/wallet_data` by default)
Owner::ListWallets -> list wallet directories

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For core, node, wallet and infrastructure proposals: Does this feature exist in other projects and what experience have their community had?
- For community, ecosystem and moderation proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other projects.

Note that while precedent set by other projects is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that Grin sometimes intentionally diverges from common project features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the project and ecosystem as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how the this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.

# References
[references]: #references

This is a sections for references such as links to other documents or reference implementations
