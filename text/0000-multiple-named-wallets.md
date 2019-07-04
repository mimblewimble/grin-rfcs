- Title: multiple-named-wallets
- Authors: [Michael Cordner](mailto:yeastplume@protonmail.com)
- Start date : June 26th, 2019
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

Modify the wallet's data structure to allow for support of multiple named wallets within a top-level directory. Provide OwnerAPI functions to allow for querying existing wallets and switching the active wallet.

# Motivation
[motivation]: #motivation

Grin's wallet currently assumes a single instance of seed/data, which is limiting from both a user's and wallet developer's perspective. Much wallet software (e.g. Electrum) provides end-users with the option to create and operate on multiple named wallets contained within a single directory, and adding this capability to the core Grin wallet's APIs will afford developers more flexibility.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

TBD

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

`Legacy` wallets will retain their existing structure, so as to minimize backwards compatibility concerns. The wallet should detect whether wallet data is already present directly in the top-level directory,
and if so assume that the wallet named 'default' refers to that directory.

A wallet created before this change that has new named wallets added will contain this directory structure:

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

Please note that the new API functions defined in [0000-full-wallet-lifecycle-rfc](#) contain additional arguments in certain API functions in anticipation of this feature.

### Functions with changed behaviour

* `OwnerAPI::create_wallet(name: Option<String>, mnemonic: String, password: String) -> Result<(), libwallet::Error>`
    - Now consideres the `name` argument, creating the wallet in a subdirectory of the top-level directory.
* `OwnerAPI::open_wallet(name: Option<String>, password: String) -> Result<(), libwallet::Error>`
    - Considers the `name` argument as above

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

* New API functions should be implemented as additions, with the new features optional to ensure complete backwards compatibility

# Drawbacks
[drawbacks]: #drawbacks

* Simplicity of having a single wallet instance

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

# Prior art
[prior-art]: #prior-art

# Unresolved questions
[unresolved-questions]: #unresolved-questions

# Future possibilities
[future-possibilities]: #future-possibilities

# References
[references]: #references


