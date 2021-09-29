- Title: multiple-named-wallets
- Authors:[Sheldon Thomas](mailto:sheldon.thomas@me.com), [Michael Cordner](mailto:yeastplume@protonmail.com)
- Start date : June 26th, 2019
- RFC PR: Edit if merged:
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

Modify the wallet's data structure to allow for support of multiple named 
wallets within a top-level directory.

Provide Owner API functions to allow for querying existing wallets and opening a
specific wallet by name.

# Motivation
[motivation]: #motivation

Grin's wallet currently assumes a single instance of the seed/data, which is 
limiting from both a user's and wallet developer's perspective. Much wallet 
software (e.g. Electrum) provides end-users with the option to create and operate 
on multiple named wallets contained within a single directory, and adding this 
capability to the core Grin wallet's APIs will afford developers more flexibility.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

The primary motivation here is developer flexibility. A hosted service provider
might wish to create a grin wallet for each of their users and perform certain
operations with that wallet, such as creating Slatepacks and finalizing
transactions. With the current wallet owner API, a system developer must
continually keep calling set_top_level_directory to change wallet data directory
to use more than one wallet. Native support for multiple named wallets would
solve this.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Wallet Backend Structure and Multiple Wallets

Currently, the default wallet data directory assumes a single wallet instance, and
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

`wallet_data` contains the seed and database for a single instance of the wallet, 
with the path to the wallet's data directory configured in `grin_wallet.toml`.

The structure of the wallet's data files will be changed to allow multiple 
wallet instances, with each named wallet contained in a directory named according 
to name of the wallet provided by the end user. Current single-instance wallet 
data will be placed into a directory called `default`, and the API and all wallet 
software will assume the default directory is to be used if no name is provided.

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

If a data directory is not provided in a wallet API call, it will be assumed to 
be operating on the `default` wallet.

There should only be a single `grin-wallet.toml` file in the wallet's data 
directory. Its configuration settings are global to all wallet instances.

### Data migration

`Legacy` wallets will retain their existing structure, so as to minimize 
backwards compatibility concerns. The wallet should detect whether wallet data 
is already present directly in the top-level directory, and if so assume that 
the wallet named 'default' refers to that directory.

A wallet created before this change that has new named wallets added will 
contain this directory structure:

```
~/.grin/main/
   grin-wallet.log
   grin-wallet.toml
   wallet_data/
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

## New API Functions and Functionality Changes

Please note that the new API functions defined in [0004-full-wallet-lifecycle-rfc](/text/0004-full-wallet-lifecycle.md) 
contain additional arguments in certain API functions in anticipation of this feature.

### Functions with changed behaviour

* `OwnerAPI::create_wallet(name: Option<String>, mnemonic: Option<ZeroingString>, 
        password: String) -> Result<(), libwallet::Error>`
    - Now considers the `name` argument, creating the wallet in a subdirectory of the top-level directory.

* `OwnerAPI::open_wallet(name: Option<String>, password: ZeroingString, useMask: bool) 
        -> Result<Option<SecretKey>, libwallet::Error>`
    - Considers the `name` argument, opening the wallet specified or the default
      wallet if name is `None`
    - Return libwallet::Error if `name` is specified but there is not a wallet
      with a matching name

* `OwnerAPI::delete_wallet(name: Option<String>) -> Result<(), libwallet::Error>`
    - Considers the `name` argument as above, deleting a wallet with the name
      specified
    - Return libwallet::Error if `name` is specified but there is not a wallet
      with a matching name
    - Does allow to delete the default wallet if `None` is passed for `name`
      argument
    - No variation of the command exists to delete all wallets. A user wishing
      to delete every wallet must do so individually by name

* `OwnerAPI::close_wallet(name: Option<String>) -> Result<(), libwallet::Error>`
    - Considers the `name` argument as above, closing a wallet session with a
      given name
    - Return libwallet::Error if `name` is specified but there is not a wallet
      with a matching name

* `OwnerAPI::get_mnemonic(name: Option<String>, password:ZeroingString) -> Result<ZeroingString,
        libwallet::Error>`
    - Considers the `name` argument as above, returning the mnemonic for the
      wallet with a given name
    - Return libwallet::Error if `name` is specified but there is not a wallet
      with a matching name

### New Functions

* `OwnerAPI::list_wallets() -> Result<Vec<String>, libwallet::Error>`
    - list created wallets (i.e. wallet subdirectory names from the top-level system wallet directory).

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

* New API functions should be implemented as additions, with the new features 
  optional to ensure complete backwards compatibility

* Note: this implementation omits a `switch_wallet` type function which switches
  an open connection from one wallet to another. Instead the API consumer should
  use `close_wallet` and then `open_wallet` with the subsequent wallet name.

# Drawbacks
[drawbacks]: #drawbacks

* Simplicity of having a single wallet instance, however passing a Null or empty
  string for the wallet name to open_wallet API will use the default wallet,
  thus ensuring a high level of backwards compatibility.
* By not adding the complexity of supporting a `switch_wallet` API - we may be
  causing some added overhead onto API consumers. Switching wallets will require
  calling `close_wallet` and then `open_wallet` to the new wallet. 

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

* The only alternative would be maintaining the current wallet design where only
  one wallet is supported and the wallet names don't matter. Expecting
  developers to use the set_top_level_directory API isn't tenable.

# Prior art
[prior-art]: #prior-art

* Grin++ supports multiple wallets
* Bitcoin Core supports multiple wallets since 2017 [1]

# Unresolved questions
[unresolved-questions]: #unresolved-questions

* Should there be a way for delete_wallet to delete *all* wallets in a single
  command?
* Does the conveniance of a `switch_wallet` API outweigh the added
  implementation complexity?

# Future possibilities
[future-possibilities]: #future-possibilities

# References
[references]: #references

1. [What's new in Bitcoin Core v0.15](https://bitcointechtalk.com/whats-new-in-bitcoin-core-v0-15-part-4-7c01c553783e)

