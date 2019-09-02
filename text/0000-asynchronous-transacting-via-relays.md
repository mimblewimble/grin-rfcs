# 0000-asynchronous-transacting-via-relays.md
Title: asynchronous-transacting-via-relays<br />
Authors: DavidBurkett<br />
Start date: September 2, 2019<br />

# Summary
Builds on 0000-online-transacting-via-tor to provide a fallback mechanism for sending to peers who are not currently accessible (offline or no TOR connection).

A limited subset of peers elect to be “relay nodes” which are responsible for storing encrypted slates for a limited period of time and serving them to peers on request.

# Motivation
There are various reasons why 24/7 uptime is not possible for all wallets, so a way for users to still receive transactions when they’re offline is valuable. This proposal provides a way of implementing such a service in a decentralized and private way.

# Description
## Community-level explanation
End-users should be given the option to use the relay system in cases where the receiving party cannot be contacted directly. Although for most cases this is the desired behavior, it may not be desirable in some situations, such as when full privacy is required, or when limited bandwidth is available. For this reason, the fallback mechanism should be optional, not mandatory.

## Reference-level explanation
To submit an asynchronous transaction:
1. Encrypt the slate (see Encryption/Decryption below) and attach the current time and a reasonable timeout (no more than 24 hours).
2. Calculate and generate the appropriate amount of PoW.
3. Using the dandelion protocol, "stem" the transaction to a relay node.
4. Each stem relay node should check the PoW, and ban any node that submits a tx below the requirements (allow a small buffer).
5. At the fluff stage, the tx should be broadcast to all peers, not just relay nodes, and every peer should check the PoW and ban if necessary.
6. Nodes with subscribed wallets should try to decrypt the transaction to determine if it belongs to them.
7. Relay nodes should store the transaction until it expires.

To check relays for your transactions:
TODO: Finish this

### Relays
A new feature flag should be added and advertised by all nodes willing to act as relays by storing and serving encrypted slates to allow asynchronous transaction building.

These nodes should expect higher storage requirements, although we may be able to support an adjustable maximum size of the encrypted data pool. These nodes will also have higher bandwidth requirements, and the network benefits if they support high upload speeds.
 
### Encryption/Decryption
Slates can be encrypted using a modification of the “Basic Stealth Address Protocol(BSAP).” 

Given a receiver public key (B=b*G) and an ephemeral sender public key (R=r*G), the sender and receiver can both computer a shared secret c=H(r*b*G)=H(r*B)=H(R*b). A partial slate can be encrypted with symmetric key c to produce the encrypted slate(ENC), and all a recipient needs to identify the slate as its own is R and the destination address (C=c*G).

`Data=(R, C, ENC)`

For improved privacy, when receiver creates response, they should generate and use a new ephemeral keypair for calculating a shared secret, rather than reusing (b, B).

### Spam/Flooding Prevention
The biggest issue with a private, decentralized relay system is, without the option of charging fees for transacting, it’s difficult to prevent spam/flooding attacks. The best option we have is PoW, which must be high enough to be useful, but low enough to avoid being an inconvenience to the user. It’s a balancing act that is hard to get right, and even harder to maintain. Rather than requiring constant code changes to support changing needs, we can do several things to ensure the PoW requirements are automatically adjusted to our needs:

1. A minimum PoW requirement based on the network hashrate. As an example, there is currently an equivalent of ~100K 1080Ti's mining c31. If we choose a minimum PoW requirement of 0.000001% of the hashrate, then a device with a 1080Ti can generate the PoW requirement in approximately .1 seconds, while less capable devices could generate the solution in just a few seconds. In this scenario, if a node decides to spam the network, they would be giving up approximately 0.000006 grins in mining profit for every spam transaction created.
2. Adjust PoW requirements based on the size of the encrypted data, the requested timeout length, and the number of transactions stored by relays in the last X minutes. To support the last requirement, peers should periodically advertise their PoW requirements.

### P2P Protocol
TODO: New message for submitting partial txs (using dandelion), message for receiving stored txs via time-range and potential channel/sharding mechanism (reduces privacy).

# Drawbacks
* Relay nodes are trusted to provide all slates, with no easy way of verifying.
* When network usage is low, relay nodes could use certain timing attacks to identify the IP address of either the sender or receiver of a transaction, and in rare cases can even identify both.

# Unresolved questions
* Will the PoW requirement be sufficient to prevent spam and flooding attacks?
* Should all nodes be relays by default for maximum privacy, or should it be limited to just a small subset of nodes?

# Future possibilities
* After downloading all encrypted slates from a relay, it’s possible to use a bloom filter or similar structure to check with other relays to download any transactions that, for malicious or benign reasons, were not provided by the first relay.
* A channel/sharding system could be used to limit how many transactions must be downloaded from a relay, at the cost of a reduction in privacy.
