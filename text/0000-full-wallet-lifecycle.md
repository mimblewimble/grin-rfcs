- Title: full-wallet-lifecycle
- Authors: [Michael Cordner](mailto:yeastplume@protonmail.com)
- Start date : June 26th, 2019
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

Increase the scope of the Grin Wallet's Owner API to support full wallet lifecycle functions.

# Motivation
[motivation]: #motivation

Grin Wallet's APIs currently provides functions for transacting and querying the contents of the wallet. However, several pieces of functionality
around wallet creation and seed/password management are not included within the API. This means that any consumers of the API will either expect their users
to initialize the wallet manually before the APIs can be used, or provide custom management for wallet lifecycle functions.

The Wallet APIs are intended to be the foundation upon which community-created wallets should be built, and the job of a wallet creator is made far
more difficult by the absence of wallet creation and seed management functions within the API. Ideally, it should be the case that a wallet can
be instantiated and managed solely via the Owner API.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

From an end-user perspective, (i.e. end-users of community wallets that use the wallet API,) this change should be transparent.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Wallet Initialization

Currently, wallet data does not exist until the user runs `grin-wallet init`. The `init` command creates `grin-wallet.toml`,
in the `~/.grin/main` directory (or `~/.grin/floonet`, or the current directory via the `-h` flag), prompts the user for a password,
creates a seed file, stores the resulting data files in the directory specified in `grin-wallet.toml` (`~/.grin/main/wallet_data` by default)
and initialises the lmdb database.

It should be possible to run `grin-wallet owner_api` or invoke the API directly from a linked binary without having instantiated a wallet.

## New API Functions

* `OwnerAPI::set_wallet_directory(dir: String) -> Result<(), libwallet::Error>`
    - On API startup, it's assumed the top-level wallet data directory is `~/.grin/main/wallet_data` (or floonet equivalent)
    - Set the top-level system wallet directory from which named wallets are read. Further calls to lifecycle functions will use this wallet directory
* `OwnerAPI::create_config(data_dir: Option<String>, config_overrides: Option<GlobalWalletConfig>) -> Result<(), libwallet::Error>`
    - Outputs a `grin-wallet.toml` file into current top-level system wallet directory
    - Optionally takes wallet configuration structure to override defaults in the grin-wallet.toml file
* `OwnerAPI::open_wallet(name: Option<String>, password: String) -> Result<(), libwallet::Error>`
    - Opens the wallet and sets it as the 'active' wallet. All further API commands will be performed against this wallet.
    - The 'name' argument is included for future use, anticipating the inclusion of multiple wallets and seeds within a single top-level wallet directory.
* `OwnerAPI::create_wallet(name: Option<String>, mnemonic: String, password: String) -> Result<(), libwallet::Error>`
    - Creates and initializes a new wallet
    - Initializes seed from given mnemonic if given, random seed otherwise
    - Should error appropriately if the wallet already exists
    - The 'name' parameter is included for future use as in `open_wallet` above.
* `OwnerAPI::get_mnemonic() -> Result<String, libwallet::Error>`
    - Returns the mnemonic from the active, (open) wallet
* `OwnerAPI::change_password(old: String, new: String) -> Result<(), libwallet::Error>`
    - Changes the password for the open wallet. This will essentially:
        - Close the wallet instance
        - Confirm the existing seed can be opened with the given password
        - Regenerate the `wallet.seed` file with the new password
        - Re-open the wallet instance
        - (Should this just operate on closed wallets instead?)
* `OwnerAPI::delete_wallet(name: Option<String>, password: String) -> Result<(), libwallet::Error>`
    - Dangerous function that removes all wallet data
    - name argument reserved for future use

### Use cases

* First time use (API Case)
   - run `grin-wallet owner_api`
   - top-level data directory is set to `~/.grin/main/wallet_data` (but nothing is yet written)
   - `create_config` called to create `grin-wallet.toml`
   - `create_wallet` called (with name == None) and seed to create a new wallet and seed, wallet data is created and initialized

* Recover from seed
   - As above, except call `create_wallet` with mnemonic seed instead

### Implementation notes

Although this document doesn't attempt to outline implementation, a few notes to consider for the implementor:

* Currently, the code that deals with wallet initialization and seed management sits outside the wallet APIs, in the `impls` crate, (denoting they're implementation specific). The implementation should attempt to refactor traits from these hard implementations into a new interface, similar to the existing WalletBackend and NodeClient interfaces (WalletLifecycleManager, for instance). The implementation within `impls` will then become an implementation of that trait, and can be substituted by wallet authors with their own implementations.
* The implementation period of this RFC may be a good time to remove the BIP32 specific code out from Grin core into the wallet or into a separate rust crate (probably more desirable).
* New API functions should be implemented as additions, with the new features optional to ensure complete backwards compatibility

# Drawbacks
[drawbacks]: #drawbacks

* Sending sensitive information such as passwords and mnemonics via the OwnerAPI is an obvious security concern.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

TBD

# Prior art
[prior-art]: #prior-art

TBD

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* Security implications of sending passwords and/or master seed mnemonics through the JSON-RPC API, and how to deal with this as securely as possible.
* Security implications of leaving master seed 'open' in memory (this is aleady a concern for most wallets, but there isn't a clear way to deal with this).
* Should upgrade mechanism to support multiple wallets just leave the default directory in place, to minimise the impact of disruption?

# Future possibilities
[future-possibilities]: #future-possibilities

The changes in this RFC lead the way for:

* Support for multiple wallets in a single top-level data directory
* An alternate method of command-line invocation whereby the wallet presents its own prompt instead of using single-use commands.

# References
[references]: #references

None


