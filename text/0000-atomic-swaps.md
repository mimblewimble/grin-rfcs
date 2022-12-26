
- Title: atomic-swaps
- Author: [Gene Ferneau](mailto:gene@ferneau.link)
- Start date: July 16, 2021
- RFC PR: Edit if merged [mimblewimble/grin-rfcs#83](https://github.com/mimblewimble/grin-rfcs/pull/83)
- Tracking Issue: [tbd]()

---

# Summary
[summary]: #summary

Atomic swaps are a technique originally developed to atomically exchange funds between mutually untrusting parties using the Bitcoin cryptocurrency.

The techniques can be adapted to work for the Grin cryptocurrency following guides developed by Jasper van der Maarel [\[1\]](#references). A previous version of the protocol was implemented using Hash-based Time Locked Contracts in the [grinswap](https://github.com/GrinSwap/proof-of-concept) wallet project.

To perform an atomic swap, each party commits to locking funds on their respective chains. The lock is setup to be opened in a way that immediately reveals the key to the other party for their lock.

A trade either happens, or fails completely.

Achieving atomicity is done through a series of cryptographically constructed contracts. The contracts use time-locks to allow for recovering from a failure condition (e.g. other party ceases communication).

Multi-party outputs are generated to ensure that spending them requires the approval of both parties. The original multi-party output is considered the funding transaction, gets partially signed during the protocol, and fully signed after successful protocol setup. Each swap party constructs their own funding transaction.

Dependent transactions, `Grin Refund` and `Grin Success`, commit to using the multi-party output as inputs, spending from the `Grin Fund` funding transaction.

Once all the dependent transactions are properly constructed and signed, each party signs and publishes their funding transaction.

# Motivation
[motivation]: #motivation

This RFC aims to specify an atomic swap protocol using adaptor signatures, accompanied by an implementation in the reference Mimblewimble wallet [grin-wallet](https://github.com/geneferneau/grin-wallet/tree/atomic). All Grin wallet implementations should be able to use this RFC to interoperate, and perform atomic swaps with each other.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

In an atomic swap, two parties agree to lock funds on one chain (e.g. Bitcoin), and interactively unlock funds on another chain (e.g. Grin). By unlocking funds on the Grin chain, the secret to unlock the funds on the Bitcoin chain is revealed. Revealing the secret this way allows the exchange to happen "atomically": when one lock is opened, so is the other lock. This atomic structure keeps one party from being able to steal funds from the other party: either both locks are opened, or neither are opened.

The reference implementation uses Bitcoin as the other chain in the swap, but potentially any other chain using Secp256k1 keys and multisignatures can work with the protocol.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Funds on the non-Grin chain need to be locked in a multisignature transaction that requires knowledge of both "atomic secrets" to be spent. A refund transaction is also needed to recover funds in case one of the parties becomes non-responsive, or the protocol needs to be aborted for another reason.

The first step is to construct and pre-sign funding transactions for each party.

Each funding transaction should have two spending conditions:

- each party signs
- one party signs after timelock

Thanks to multisignature protocols like Musig2 and ECDSA2p, we can use key aggregation to construct multisignatures that look like normal signatures.

If Alice wants to swap Grin for Bob's Bitcoin, Alice constructs a multisignature output [\[5\]](#references), and an absolute timelock to allow for recovery.

Bob also constructs a multisignature output, and an absolute timelock (`OP_CHECKLOCKTIME`) to allow for recovery.

`Grin Fund` requires knowledge of both Alice and Bob's private keys to spend, or just Alice's after a timelock expires.

`Bitcoin Fund` requires knowledge of both Alice and Bob's adaptor secret keys to spend, or just Bob's private key after a timelock expires.

Alice creates the `Grin Fund` funding transaction, and constructs `Grin Refund` and `Grin Success` transactions which spend from the funding transaction.

`Grin Success` sends Grin to Bob, uses an adaptor signature to reveal Bob's secret, and allows Alice to retrieve the Bitcoin funds.

`Grin Refund` refunds Grin to Alice, uses an adaptor signature to reveal Alice's secret, and allows Bob to retrieve Bitcoin funds using the multisignature spend path.

Below is a diagram of a successful atomic swap protocol:

| Alice | Direction | Bob |
|-------|-----------|-----|
| Create(GrinFund) | ---------> | |
| | <--------- | Create(BitcoinFund) |
| Create(GrinRefund) + AdaptSign(GrinRefund) | ---------> | |
| | <--------- | PreSign(GrinRefund) |
| Create(GrinSuccess) | ---------> | |
| | <--------- | AdaptSign(GrinSuccess) |
| PreSign(GrinSuccess) | ---------> | |
| | <--------- | PreSign(GrinFund) |
| Sign(GrinFund) + Publish(GrinFund) | | |
| | | Sign(GrinSuccess) + Publish(GrinSuccess) |
| RecoverAtomic(GrinSuccess) | | |
| Sign(BitcoinFund) + Publish(BitcoinFund) | | |


## Atomic secrets
[atomic-secrets]: #atomic-secrets

In the case of Grin, atomic secrets are just Secp256k1 private keys used to unlock funds on the other chain. Let's say Alice knows atomic secret `nA`, and Bob knows atomic secret `nB`.

In the refund transaction, Alice (with locked Grin funds) will reveal her secret `nA` to allow Bob to recover his locked funds on the other chain.

In the `Grin Success` transaction, Bob and Alice create a multisignature Grin transaction which reveals Bob's secret `nB` to Alice. Alice can then recover the funds locked on the other chain.

Atomic IDs are identifiers used to derive atomic secrets using the keychain. Atomic IDs are formatted as follows:

```
0x3 | 'mwatomic' | 32-bit ID | 0x0 0x0 0x0 0x0
```

The 32-bit ID can be expanded to 64-bits if necessary, and doesn't need to be random (it can be a simple counter). However, an ID can only be used once.

Currently, output blinding factors and signature nonces use the key ID of the inputs and outputs, which is derived from the current child key path, or next available key path.

Atomic swaps still use the default methods for deriving blinding factors and signature nonces. The atomic IDs are domain separated to help ensure they don't collide with normal blinding factors and signature nonces.

Atomic secrets must only be used **once**. Reuse potentially results in loss of funds, if the other party has the previously used secret.

Without automated defenses, it may be possible for a user to mistakenly reuse a secret. A malicious user could then recover funds on the other chain without completing the atomic swap protocol.

Regular blinding factors and signature nonces use a strictly increasing counter to prevent deriving the same factors more than once.

Atomic secrets use a similar strictly increasing counter, with the additional protection of using the domain separated prefix.

## Adaptor signatures
[adaptor-signatures]: #adaptor-signatures

Adaptor signatures were originally invented by Andrew Poelstra [\[2\]](#references). Adaptor signatures use additive properties of Schnorr signatures to commit to a secret, then later reveal it by creating a full Schnorr signature over the same message.

## Multisignature outputs
[multisignature-outputs]: #multisignature-outputs

Atomic swap transactions rely on multisignature outputs, which are normal Grin outputs owned by multiple parties. There is a [draft multisignature output specification](https://github.com/mimblewimble/grin-rfcs/pull/85) detailing 2-of-2 multisignature outputs.

Multisignature outputs are different from multisignature kernels. In a multisignature kernel, both parties supply their own inputs and outputs, and collaborate to construct the multisignature kernel. Multisignature outputs are used to create multisignature kernels in exactly the same way as single-signature outputs.

Multisignature outputs require both (all) parties to collaboratively build both the Pedersen commitment to the output value, and the rangeproof. Once fully constructed, multisignature outputs function the same as normal, single-signature outputs. To an outside observer, multisignature outputs are the same as single-signature outputs.

## Slate v5
[slate]: #slate

In addition to the slate changes in the multisignature specification, there are a number of changes needed specifically for atomic swaps. Atomic swaps depend on the changes in multisignature slate format, but multisignature outputs do not depend on the changes for atomic swaps.

There are additional variants to add to the `SlateState` enum (`Name => Serialized`):

- Atomic1 => "A1"
- Atomic2 => "A2"
- Atomic3 => "A3"
- Atomic4 => "A4"

The `ParticipantData` struct needs the additional optional field:

- `public_nonce`: Secp256k1 public key

`ParticipantData` is serialized according to the specification in multisignature ouptuts RFC, repeated here for convenience:

- Encoding the presence of optional fields in the optional flag (currently 0 or 1)
  - 1-bit set indicates partial signature (`part`) present
  - 2-bit set indicates atomic public key (`atomic`) present
  - 4-bit set indicates partial commit (`part_commit`) present
  - 8-bit set indicates Tau X key (`tau_x`) present
  - 16-bit set indicates Tau One key (`tau_one`) present
  - 32-bit set indicates Tau Two key (`tau_two`) present
- `SigWrap` is serialized in the following order
  - length (number of ParticipantData entries)
  - each ParticipantData entry
- `ParticipantData` is serialized in the following order
  - optional flag (unsigned 8-bit integer)
  - blinding factor (Secp256k1 public key, compressed)
  - nonce (Secp256k1 public key, compressed)
  - atomic public key (Secp256k1 public key, compressed) (if present)
  - partial signature (Secp256k1 signature) (if present)
  - partial commit (Pedersen commitment) (if present)
  - tau x key (Secp256k1 secret key) (if present)
  - tau one key (Secp256k1 public key, compressed) (if present)
  - tau two key (Secp256k1 public key, compressed) (if present)

## Protocol rounds
[protocol-rounds]: #protocol-rounds

In Grin, let `kB` be Bob's random kernel nonce, `rB` be Bob's random blinding factor, `nB` be Bob's atomic secret. Let `kA` and `rA` be Alice's random kernel nonce and blinding factor.

Alice and Bob must perform an atomic swap refund transaction before the success transaction is accepted into the blockchain. Since both transactions spend the same output, it is not possible to post both transactions (double-spending).

Multisignature outputs are indicated by boxes with two owners, e.g. `Alice + Bob`.

The multisignature Bitcoin output with `secretA + secretB` is simply signed under the aggregate private key (`secretA + secretB`), and the signature is verified against the aggregate public key (`(secretA + secretB) * G`).

An illustration of the protocol:

```
+----------+                  +-------------+                     +----------+
| 87k Grin |  - Grin Fund ->  | 87k Grin    |   - Grin Refund ->  | 87k Grin |
| Alice    |                  | Alice + Bob |     2 days          | Alice    |
+----------+                  +-------------+     reveals         +----------+
                                                  secretA
                                     | 
                                     | 
                                     | 
                                     |                         
                                     |                   +----------+
                                     +- Grin Success ->  | 87k Grin |
                                        reveals          | Bob      |
                                        secretB          +----------+


           
+-------+                     +-------------------+
| 1 BTC |  - Bitcoin Fund ->  | 1 BTC             |
| Bob   |                     | secretA + secretB |
+-------+                     +-------------------+
```

The following is a description of the `Grin Success` transaction rounds.

The transaction is built like a typical Grin transaction, except that it only spends the multisignature output from the Grin Fund transaction. This results in a 1-in-1-out transaction, minus fees.

It is up to atomic swap participants if the initiator (Alice) or receiver (Bob) pays for the fees on the atomic swap transaction. If Alice pays the fees, the fee needs to be added to the agreed value when creating the multisignature output. If Bob pays the fee, the fee is simply subtracted from the multisignature output during the atomic swap transaction.

*TBD*: Is it safe to supply additional inputs (other than the multisignature output) to pay for fees? Currently the atomic swap implementation in `grin-wallet` restricts to only spending the multisignature output.

1. `init_atomic_swap`

In the first round, Alice selects and verifies the multisignature output from the Grin Fund transaction, and sends her public random kernel nonce and blinding factor (`kA*G` and `rA*G`) to Bob.

Alice's coin selection works similar to normal Grin coin selection, except that only valid multisignature outputs are considered for selection.

Multisignature output validation includes checking for proper ID, and that the output is unspent.

Alice also includes the multisignature output ID in the slate, so that Bob can select the same output.

2. `receive_atomic_swap`

Bob selects and verifies the stored multisignature output using the multisignature output ID included in the slate.

Bob creates an adaptor signature as follows (where `e` is the kernel message):

```
sr' = kB + nB + rB*e
```

Bob sends `sr'` as his partial signature, and public atomic secret `nB*G` in the second round of the swap.

3. `countersign_atomic_swap`

Alice can verify the adaptor signature by supplying Bob's public atomic secret to the Schnorr verification algorithm as an extra nonce (this capability exists in Grin's fork of `libsecp256k1`). Alice stores the `s` component (`s = sr'[32:]`) to recover the atomic secret during swap finalization.

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

During a swap, a number of conditions can lead to a protocol abort. The `Grin Refund` transaction is similarly structured to the `Grin Success` protocol rounds, but Bob initiates the transaction instead of Alice.

`Grin Refund` uses a `lock_height` kernel for an absolute timelock set to a reasonable time in the future (2 days in the example diagram).

`Grin Refund` reveals Alice's atomic secret (`nA`). Bob can then recover his funds on the other chain.

The funding transaction on the other chain can also use a timelock to abort the protocol, if timelocks exist on the other chain. Funds are returned to their original owners.

### Recovering funds on the other chain
[recovering-funds]: #recovering-funds

The `Bitcoin Fund` transaction in the diagram is a mutlisignature transaction, requiring a signature under both Alice's and Bob's atomic secrets (`nA` and `nB`). Since Alice or Bob will know both secrets when recovering the funds, they can simply aggregate their keys, and sign using a normal single key signature.

So, the `Bitcoin Fund` transaction will use the aggregate public key `nA*G + nB*G`.

The transaction can also optionally include a timelock refunding Bob after a given timeout (e.g. 4 days). For chains that do not have timelocks, the timelock is not strictly necessary, but Bob's funds could potentially be locked indefinitely.

Timelocks allow for a "fail open" condition, where Bob is able to retrieve his funds, even if Alice does not post the `Grin Refund` transaction after the timelock expires.

## Secret recovery
[secret-recovery]: #secret-recovery

With the kernel excess commitment, Alice can recover the transaction kernel after the transaction is posted to the blockchain. Alice can then use her partial signature, and the finalized kernel excess signature to recover Bob's partial signature.

Alice subtracts Bob's partial signature from his adaptor signature to recover his atomic secret `nB`:

```
nB = sr' - sr
   = (kB + nB + rB*e) - (kB + rB*e)
   = nB + (kB - kB) + (rB*e - rB*e)
```

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Is there any way to reduce the amount of rounds required per atomic swap transaction (currently 4 rounds each, 8 total)?

# Future possibilites

Ruben Somsen developed a protocol called Succinct Atomic Swaps [\[3\]](#references) that could be adapted to work with Grin. A separate Grin RFC PR [#80](https://github.com/mimblewimble/grin-rfcs/pull/80) is open, and currently in `Draft` status detailing Succinct Atomic Swap specification for Grin. The main advantage of Succinct Atomic Swaps is a recovery mechanism through three additional transactions `Revoke`, `Refund #2` and `Timeout`.

`Revoke` is a multisignature output transaction with an absolute timelock (`lock_height` in Grin). `Refund #2` and `Timeout` spend the multisignature output created by `Revoke`. `Refund #2` has either an absolute (`lock_height`) or relative (`NRD`) timelock that is longer than the timeout on `Revoke`. `Refund #2` refunds the Grin funds to Alice, and reveals `secretA` to Bob. `Timeout` has an absolute (`lock_height`) or relative (`NRD`) timelock longer than `Revoke` and `Refund #2`, and releases the Grin funds to Bob, without revealing Bob's `secretB`.

Thus, a situation can happen where Bob receives the Grin funds, but the Bitcoin funds are locked forever (Alice has no way to recover them).

Ruben Somsen's protocol allows for a 2-transaction version, where only two on-chain transactions are posted (excluding funding transactions). This means one Grin transaction, and one transaction on the other chain (e.g. Bitcoin). The 2-transaction protocol requires using watchtowers, and for the parties to be online for the duration of the swap. It may not be worth the additional complexity, considering it only saves one on-chain transaction, and Grin transactions are currently not that expensive. If Grin fees were ever to reach/exceed Bitcoin's, it may be worth reconsidering.

Somsen's original design uses relative timelocks, which could be implemented on Grin using No Recent Duplicate kernels [\[4\]](#references) when they are activated on Grin mainnet. This would allow for a more automated way to post the `Refund #2` and `Timeout` transactions.

# References
[references]: #references
- \[1\] [Grin docs for Atomic Swaps](https://docs.grin.mw/wiki/transactions/contracts/#atomic-swap)
- \[2\] [Scriptless Scripts](https://download.wpsoftware.net/bitcoin/wizardry/mw-slides/2018-05-18-l2/slides.pdf)
- \[3\] [Succinct Atomic Swaps](https://gist.github.com/RubenSomsen/8853a66a64825716f51b409be528355f)
- \[4\] [NRD Kernels](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0013-nrd-kernels.md)
- \[5\] [Multisignature Outputs](https://github.com/GeneFerneau/grin-rfcs/blob/multisig/text/0000-multisignature-outputs.md)
