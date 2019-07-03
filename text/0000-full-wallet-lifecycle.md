- Title: full-wallet-lifecycle
- Authors: [Michael Cordner](mailto:yeastplume@protonmail.com)
- Start date : June 26th, 2019
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

Increase the scope of the Grin Wallet's Owner API to support all wallet lifecycle functions. Wallet creation, seed management, (TBD)

# Motivation
[motivation]: #motivation

Grin Wallet's APIs currently provides functions for transacting and querying the contents of the wallet. However, several pieces of functionality
around wallet creation and seed/password management are not included within the API. This means that any consumers of the API will either expect their users
to initialize the wallet manually before the APIs can be used, or provide custom management for wallet lifecycle functions.

The Wallet APIs are intended to be the foundation upon which community-created wallets should be built, and the job of a wallet creator is made far
more difficult by the absence of wallet creation and seed management functions within the API. Ideally, it should be the case that a wallet can
be instantiated and managed solely via the Owner API.

In order to achieve this, several other pieces of functionality will need to in place. These are outlined in detail below, but as a summary:

* Support for multiple wallets within a single data directory
* Change the model for the command-line wallet from single-use commands to an internal prompt, keeping the wallet instance resident between commands.

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

If a data directory is not provided in a wallet API call, it will be assumed to be operating on the `default` wallet.

There should only be a single `grin-wallet.toml` file in the wallet's data directory. Its configuration settings are global to all wallet
instances.

### Data migration

The target version of the wallet will contain a function to migrate existing wallets from the current data structure to the new,
essentially moving `wallet_data/wallet.seed`, `wallet_data/db/` and `wallet_data/saved_txs/` into `wallet_data/default/wallet.seed` etc.
(Should this be performed within an API call for consistency, or should existing files just be left alone?)

## Wallet Initialization

Currently, wallet data does not exist until the user runs `grin-wallet init`. The `init` command creates `grin-wallet.toml`,
in the `~/.grin/main` directory (or `~/.grin/floonet`, or the current directory via the `-h` flag), prompts the user for a password,
creates a seed file, stores the resulting data files in the directory specified in `grin-wallet.toml` (`~/.grin/main/wallet_data` by default)
and initialises the lmdb database.

It should be possible to run `grin-wallet owner_api` or invoke the API directly from a linked binary without having instantiated a wallet.

## Wallet Runtime Instantiation

Currently, wallet instantiation works differently depending on whether the wallet is invoked via the APIs or via the command line.

1. In the command line case, each wallet command is a separate invocation. Command line invocation will ask for a password, decrypt the master seed and initialize the wallet with the descrypted seed. It will then perform the desired function and return, zeroing memory and exiting the process.

1. In the case of a listening API (owner or foreign), the password is given once and the wallet seed is decrypted. The wallet instance is then kept in memory by the handling thread, and re-used for each Foreign or Owner API call.

In order to retain consistency and provide a framework through which the wallet can be run and lifecycle API functions can be called without wallet data actually being present, we propose that the operational model of the command-line wallet be changed to point 2 above. When the command-line wallet is first invoked, it will essentially present nothing other than a prompt. The user can then enter commands to create a wallet, instantiate a particular wallet, change the active wallet, etc. via the wallet's command prompt.

Note that (wallet713)[https://github.com/vault713/wallet713] by vault713 already works exactly as this feature is described. Pending approval by vault 713, we propose merging the relevant code from wallet713 directly into the Grin command line wallet.

Note this will mean significant changes from an end-user perspective on the relevant wallet release, as the existing command line commands will all be replaced with internal equivalents. There will also be additional end-user lifecycle commands that call new wallet lifecycle APIs.

## New API Functions

* `OwnerAPI::set_wallet_directory(dir: String) -> Result<(), libwallet::Error>`
    - On API startup, it's assumed the top-level wallet data directory is `~/.grin/main/wallet_data` (or floonet equivalent)
    - Set the top-level system wallet directory from which named wallets are read. Further calls to lifecycle functions will use this wallet directory
* `OwnerAPI::create_config(data_dir: Option<String>, config_overrides: Option<GlobalWalletConfig>) -> Result<(), libwallet::Error>`
    - Outputs a `grin-wallet.toml` file into current top-level system wallet directory
    - Optionally takes wallet configuration structure to override defaults in the grin-wallet.toml file
* `OwnerAPI::list_wallets() -> Result<Vec<String>, libwallet::Error>`
    - list created wallets (i.e. wallet subdirectory names from the top-level system wallet directory).
* `OwnerAPI::create_wallet(name: Option<String>, mnemonic: String, password: String) -> Result<(), libwallet::Error>`
    - Creates and initializes a new wallet in the subdirectory specified by `name`, `default` if None
    - Initializes seed from given mnemonic if given, random seed otherwise
    - Should error appropriately if the wallet subdir already exists
* `OwnerAPI::open_wallet(name: Option<String>, password: String) -> Result<(), libwallet::Error>`
    - Opens the specified wallet and sets it as the 'active' wallet. All further API commands will be performed against this wallet.
* `OwnerAPI::get_mnemonic() -> Result<String, libwallet::Error>`
    - Returns the mnemonic from the active, (open) wallet
* `OwnerAPI::change_password(old: String, new: String) -> Result<(), libwallet::Error>`
    - Changes the password for the open wallet. This will essentially:
        - Close the wallet instance
        - Confirm the existing seed can be opened with the given password
        - Regenerate the `wallet.seed` file with the new password
        - Re-open the wallet instance
        - (Should this just operate on closed wallets instead?)

### Use cases

* First time use
   - run `grin-wallet` (or launch OwnerAPI for web wallet)
   - top-level data directory is set to `~/.grin/main/wallet_data` (but nothing is yet written)
   - Optionally config data directory
   - `create_config` called to create `grin-wallet.toml`
   - `create_wallet` called with new wallet name and seed to create a new wallet and seed
   - new wallet subdirectory is created and initialized

* Recover from seed
   - As above, except call `create_wallet` with mnemonic seed instead

### Implementation notes

Although this document doesn't attempt to outline implementation, a few notes to consider for the implementor:

* Currently, the code that deals with wallet initialization and seed management sits outside the wallet APIs, in the `impls` crate, (denoting they're implementation specific). The implementation should attempt to refactor traits from these hard implementations into a new interface, similar to the existing WalletBackend and NodeClient interfaces (WalletLifecycleManager, for instance). The implementation within `impls` will then become an implementation of that trait, and can be substituted by wallet authors with their own implementations.
* The implementation period of this RFC may be a good time to remove the BIP32 specific code out from Grin core into the wallet or into a separate rust crate (probably more desirable).
* New API functions should be implemented as additions, with the new features optional to ensure complete backwards compatibility
* The implementation should likely be split up into separate PRs (possibly across multiple releases), in rough order:
    * Support for multiple wallets, and upgrade mechanism
    * Trait refactoring and new API function implementation
    * (wallet713-based) CLI Rework, including calls to new API functions

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

* Security implications of sending passwords and/or master seed mnemonics through the JSON-RPC API, and how to deal with this as securely as possible.
* Security implications of leaving master seed 'open' in memory (this is aleady a concern for most wallets, but there isn't a clear way to deal with this).
* Should upgrade mechanism to support multiple wallets just leave the default directory in place, to minimise the impact of disruption?

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

**wallet713**
- https://github.com/vault713/wallet713

