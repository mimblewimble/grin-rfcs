
- Title: succinct-atomic-swaps
- Author: [Gene Ferneau](mailto:gene@ferneau.link)
- Start date: March 21, 2021
- RFC PR: [tbd]()
- Tracking Issue: [tbd]()

---

# Summary
[summary]: #summary

Succinct atomic swaps [\[1\]](#references) are a technique developed by Ruben Somsen to atomically exchange funds between mutually untrusting parties using the Bitcoin cryptocurrency.
The techniques can be adapted to work for the Grin cryptocurrency following guides developed by Jasper van der Maarel [\[2\]](#references). A previous version of the protocol was implemented
using Hash-based Time Locked Contracts in the [grinswap](https://github.com/GrinSwap/proof-of-concept) wallet project.

# Motivation
[motivation]: #motivation

This RFC aims to show a succinct atomic swap protocol using adaptor signatures, accompanied by an implementation in the reference Mimblewimble wallet [grin-wallet](https://github.com/geneferneau/grin-wallet/tree/atomic). All Grin wallet implementations should be able to use this RFC to interoperate, and perform atomic swaps with each other.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

In an atomic swap, two parties agree to lock funds on one chain (e.g. Bitcoin), and interactively unlock funds on another chain (e.g. Grin). By unlocking funds on the Grin chain, the secret to unlock the funds on the Bitcoin chain is revealed. Revealing the secret this way allows the exchange to happen "atomically": when one lock is opened, so is the other lock. This atomic structure keeps one party from being able to steal funds from the other party: either both locks are opened, or neither are opened.

The reference implementation uses Bitcoin as the other chain in the swap, but potentially any other chain using Secp256k1 keys and multisignatures can work with the protocol.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Funds on the non-Grin chain need to be locked in a multisignature transaction that requires knowledge of both "atomic nonces" to be spent. A refund transaction is also needed to recover funds in case one of the parties becomes non-responsive, or the protocol needs to be aborted for another reason.

## Atomic nonces
[atomic-nonces]: #atomic-nonces

In the case of Grin, atomic nonces are just Secp256k1 private keys used to unlock funds on the other chain. Let's say Alice knows atomic nonce `nA`, and Bob knows atomic nonce `nB`.

In the refund transaction, Alice (with locked Grin funds) will reveal her nonce `nA` to allow Bob to recover his locked funds on the other chain.

In the main transaction, Bob and Alice create a multisignature Grin transaction which reveals Bob's nonce `nB` to Alice. Alice can then recover the funds locked on the other chain.

Atomic IDs are identifiers used to derive atomic nonces using the keychain. Atomic IDs are formatted as follows:

```
0x3 | 'mwatomic' | 32-bit ID | 0x0 0x0 0x0 0x0
```

The 32-bit ID can be expanded to 64-bits if necessary, and doesn't need to be random (it can be a simple counter). However, an ID can only be used once.

Currently, output blinding factors and signature nonces use the key ID of the inputs and outputs, which is derived from the current child key path, or next available key path.

Atomic swaps still use the default methods for deriving blinding factors and signature nonces. The atomic IDs are domain separated to help ensure they don't collide with normal blinding factors and signature nonces.

Atomic nonces must only be used **once**. Reuse potentially results in loss of funds, if the other party has the previously used nonce.

Without automated defenses, it may be possible for a user to mistakenly reuse a nonce, or a malicious party to trick a user into deriving the same nonce (by supplying a duplicate atomic ID). A malicious user could then recover funds on the other chain without completing the atomic swap protocol.

Regular blinding factors and signature nonces use a strictly increasing counter to prevent deriving the same factors more than once.

Atomic nonces use a similar strictly increasing counter, with the additional protection of using the domain separated prefix.

## Adaptor signatures
[adaptor-signatures]: #adaptor-signatures

Adaptor signatures were originally invented by Andrew Poelstra [\[3\]](#references). Adaptor signatures use additive properties of Schnorr signatures to commit to a secret, then later reveal it by creating a full Schnorr signature over the same message.


## Protocol rounds
[protocol-rounds]: #protocol-rounds

In Grin, let `kB` be Bob's random kernel nonce, `rB` be Bob's random blinding factor, `nB` be Bob's atomic nonce. Let `kA` and `rA` be Alice's random kernel nonce and blinding factor.

Alice and Bob must perform an atomic swap revoke transaction before the success transaction.

The revoke transaction must be spent by either the refund or timeout transaction. An absolute timelock (`lock_height`) is added to the revoke transaction.

In Somsen's original SAS protocol, a relative timelock is added to the refund and timeout transactions. An absolute timelock also works in the 3-transaction protocol. The refund transaction has a longer timelock than the revoke transaction, and the timeout transaction has a longer timelock than the refund transaction.

In Grin, the requirement for the refund and timeout transaction to spend the revoke transaction can be achieved by using the revoke transaction's output as the input.

An illustration of the protocol (credit @tromp + @RubenSomsen):

```


              Grin Fund                    Grin Revoke                    Grin Refund
+----------+              +-------------+  1 day         +-------------+  2 days        +----------+
| 87k Grin |  --------->  | 87k Grin    |  ----------->  | 87k Grin    |  ----------->  | 87k Grin |
| Alice    |              | Alice + Bob |                | Alice + Bob |  reveals       | Alice    |
+----------+              +-------------+                +-------------+  secretA       +----------+
                                                             
                                  |                       | Grin Timeout
                                  |                       | 3 days
                                  |                       v
                                  |                         
                                  | Grin Success   +----------+
                                  +------------->  | 87k Grin |
                                    reveals        | Bob      |
                                    secretB        +----------+


           Bitcoin Fund
+-------+                  +-------------------+
| 1 BTC |  ------------>   | 1 BTC             |
| Bob   |                  | secretA + secretB |
+-------+                  +-------------------+


```

The following is a description of the `Grin Success` transaction rounds.

The transaction is built like a typical 1-in-2-out Grin transaction. The actual number of inputs could be more (maybe Alice has many small UTXOs), but the concept is the same. One output is sent to Bob, the other is the change output sent back to Alice.

1. `init_atomic_swap`

In the first round, Alice sends her public random kernel nonce and blinding factor (`kA*G` and `rA*G`) to Bob.

2. `receive_atomic_swap`

Bob creates an adaptor signature as follows (where `e` is the kernel message):

```
sr' = kB + nB + rB*e
```

Bob sends `sr'` as his partial signature, and public atomic nonce `nB*G` in the second round of the swap.

3. `countersign_atomic_swap`

Alice can verify the adaptor signature by supplying Bob's public atomic nonce to the Schnorr verification algorithm as an extra nonce (this capability exists in Grin's fork of `libsecp256k1`). Alice stores the `s` component (`s = sr'[32:]`) to recover the atomic nonce during swap finalization.

If the adaptor signature is valid, Alice computes her partial signature over the kernel message `e = SHA256(M | kB*G + kA*G)`, using `kA` and `rA` as her kernel nonce and blinding factor:

`ss = kA + rA*e`

4. `finalize_atomic_swap`

Bob verifies the partial signature, and if valid, creates his final signature:

`sr = kB + rB*e`

and adds it to Alice's partial signature:

`sr + ss = kA + kB + rA*e + rB*e`

Bob posts the transaction to the blockchain, and sends the final slate to Alice. The slate is used for Alice to recover her partial signature, which may be possible by encrypting the third round slate to both Bob and herself (TBD). Currently, in the tests Alice recovers her partial signature from the finalized slate. This is not ideal, since Bob can simply finalize/post the transaction, without sending the slate to Alice.

### Protocol abort
[protocol-abort]: #protocol-abort

During a swap, a number of conditions can lead to a protocol abort. The `Grin Revoke` transaction is similarly structured to the `Grin Success` protocol rounds, but Bob initiates the transaction instead of Alice.

`Grin Revoke` uses a `lock_height` kernel for an absolute timelock set to a reasonable time in the future (2 days in the example diagram).

The `Grin Refund` and `Grin Timeout` transactions are used to spend the `Grin Revoke` transaction.

`Grin Refund` uses a longer absolute timelock than `Grin Revoke`, refunds the locked Grin to Alice, and reveals Alice's atomic nonce (`nA`). Bob can then recover his funds on the other chain.

`Grin Timeout` uses a longer absolute timelock then `Grin Refund` and `Grin Revoke`, and releases the funds to Bob. Alice is not able to retrieve the funds on the other chain, since Bob's atomic nonce is never revealed, so is highly incentivised to not let the swap timeout.

*TBD*: Bob could potentially attack Alice with a Denial-of-Service after the `Grin Revoke` transaction is published, forcing a `Grin Timeout` abort. Should the `Grin Timeout` transaction require Bob to reveal his atomic nonce (`nB`), so Alice can claim the funds on the other chain?

### Recovering funds on the other chain
[recovering-funds]: #recovering-funds

The `Bitcoin Main Multisig` transaction in the diagram is a mutlisignature transaction, requiring a signature under both Alice's and Bob's atomic nonces (`nA` and `nB`). Since Alice or Bob will know both secrets when recovering the funds, they can simply add their nonces, and sign using a normal single key signature.

So, the `Bitcoin Main Multisig` transaction will use the aggregate public key `nA*G + nB*G`.

The transaction can also optionally include a timelock refunding Bob after a given timeout (e.g. 4 days). For chains that have multisignature support, but not timelocks, the timelock is not strictly necessary.

Timelocks do allow for a "fail open" condition, where Bob is able to retrieve his funds, even if Alice does not post the `Grin Refund` transaction after the `Grin Revoke` timelock expires.

## Nonce recovery
[nonce-recovery]: #nonce-recovery

With the kernel excess commitment, Alice can recover the transaction kernel after the transaction is posted to the blockchain. Alice can then use her partial signature, and the finalized kernel excess signature to recover Bob's partial signature.

Alice subtracts Bob's partial signature from his adaptor signature to recover his atomic nonce `nB`:

```
nB = sr' - sr
   = (kB + nB + rB*e) - (kB + rB*e)
   = nB + (kB - kB) + (rB*e - rB*e)
```

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Is there any way to reduce the amount of rounds required per atomic swap transaction (currently 4 rounds each, 8 total)?

# Future possibilites

Ruben Somsen's protocol allows for a 2-transaction version, where only two on-chain transactions are posted (excluding funding transactions). This means one Grin transaction, and one transaction on the other chain (e.g. Bitcoin). The 2-transaction protocol requires using watchtowers, and for the parties to be online for the duration of the swap. It may not be worth the additional complexity, considering it only saves one on-chain transaction, and Grin transactions are currently not that expensive. If Grin fees were ever to reach/exceed Bitcoin's, it may be worth reconsidering.

Somsen also has a variant using relative timelocks, which could be implemented on Grin using No Recent Duplicate kernels [\[4\]](#references). This would allow for a more automated way to post the refund transaction, but requires NRD kernel activation on mainnet.

# References
[references]: #references
- \[1\] [Succinct Atomic Swaps](https://gist.github.com/RubenSomsen/8853a66a64825716f51b409be528355f)
- \[2\] [Grin docs for Atomic Swaps](https://docs.grin.mw/wiki/transactions/contracts/#atomic-swap)
- \[3\] [Scriptless Scripts](https://download.wpsoftware.net/bitcoin/wizardry/mw-slides/2018-05-18-l2/slides.pdf)
- \[4\] [NRD Kernels](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0013-nrd-kernels.md)
