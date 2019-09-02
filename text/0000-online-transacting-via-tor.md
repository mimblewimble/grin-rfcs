# 0000-online-transacting-via-tor<br/>
Title: online-transacting-via-tor<br/>
Authors: DavidBurkett<br/>
Start date: September 1, 2019<br/>

# Summary
Describes a standardized addressing and communication protocol for building Grin transactions.<br/><br/>
Wallets will attempt to connect to the recipient directly to build transactions over a TOR hidden service similar to how http(s) transaction building works today.<br/><br/>
If the recipient is not online, or cannot be contacted directly, users have the option of falling back to an “offline” communication system where slates are encrypted and stored, for a limited period of time, by a subset(TBD) of nodes known as “relay nodes” (described in a future RFC).

# Motivation
Grin is unique in that it requires the sender and receiver to interact in order to transact. This presents a lot of unique challenges that most coins don’t have to deal with. There are a number of different incompatible standards for sending and receiving, resulting in confusion and headaches for many users. The hope is that the addressing mechanism described here will become the new default method for sending and receiving, deprecating several less secure and less private methods in the process.

# Description
## Community-level explanation
From an end-user perspective, there should no longer be a need to configure routers and firewalls to receive grins. Sending and receiving should feel like any other cryptocurrency, where a simple encoded address is all you need to share before receiving. No firewall or router configuration should be necessary.

## Reference-level explanation
### Addressing
Onion addresses for TOR hidden services are generated from an ed25519 public key (32 bytes), and include a checksum and a version[1]. This provides an equivalent level of security as bitcoin addresses, and can be ephemeral or permanent, depending on the user’s needs. Grin addresses should be generated in the same way as TOR addresses, although we have the flexibility to decide whether we would like to encode them using base32 (like TOR), base58 (like Bitcoin), or even a far superior encoding based on emojis.<br/><br/>
Although ed25519 is a different curve than used by the grin protocol, we can still use our HD wallets to generate deterministic ed25519 public keys (and therefore Grin addresses). We can decide to generate them from a special keychain path, or even using a different master seed [2], along with a KDF for converting those HD child keys to ed25519-compatible private keys.

### TOR Hidden Services
TOR hidden services can be used to directly serve the existing foreign APIs. When configuring TOR (whether bundled with grin, or installed separately), you would just publish a hidden service and configure TOR to forward all traffic to port 3420. This means we can continue supporting http(s) sending/receiving with no disruption, though it’s advisable to avoid sending directly over http(s) asap.In future versions of Grin, we can stop allowing non-local connections to the foreign wallet APIs.

# Drawbacks
Requires users to setup and configure TOR, or bundle it with Grin, which could be non-trivial, and could conflict with locally installed/running versions.

# Unresolved questions
* How should we encode addresses? base32, base58, emoji, etc.
* How often should addresses change? Should users manually request a new address, or should they auto expire? Should we support multiple?

# Future possibilities
The changes in this RFC lead the way for:
* Payment proofs
* Offline transacting via SBBS/Grinbox-style relay system. See “Offline Transactions” RFC

# References
[1] https://github.com/torproject/torspec/blob/87698dc1c0fa4cf2186f180a636fc7ad1c5fb5fd/rend-spec-v3.txt#L2059-L2081 <br/>
[2] https://github.com/mimblewimble/grin/blob/d918c5fe84e859290c9d09f5cfc167ed41d27bff/keychain/src/extkey_bip32.rs#L122
