
- Title: multisig-wallet
- Authors: [valdok](https://github.com/valdok)
- Start date: Dec 16, 2025

---

## Summary
Support an M/N multisig wallet.

## Motivation
The idea is to allow a wallet to be co-owned by several users, whereas a quorum of M/N is both required and sufficient to build a valid transaction (i.e. send or receive funds).


## Reference-level explanation

The core idea is based on Shamir's secret sharing (SSS) scheme, which allows the reconstruction of a secret by a quorum of M/N. And due to the homomorphic nature of ECC, the secret keys will not be calculated in a plain way. It will only be used to compute the public keys, and to calculate the signatures in an MPC ritual required to build transactions (such as Schnorr's signature, and UTXO rangeproof)

Normal wallet has a master secret (usually derived from the seed phrase), which is then used to derive various keys, including the blinding factors for coins (such as BIP44).

Multisig wallet will instead rely on SSS to generate the keys. The BIP44 can still be used in addition, to increase the security of the wallet (more about this later).


### Key concepts

Here we'll define key terms and concept
 - We'll operate on secp256k1 curve (the native for Grin).
 - **Secret key** - EC (elliptic curve) scalar
 - **Pubkey** - EC point (group element).
 - **Actor** - a user that co-owns a wallet.
 - **Address** - the Grin SlatepackAddress. Each actor will have the address for both identification, and communication (encryption of messages).
   - Note that Grin Slatepack addresses are defined over a different EC: ed25519.
 
 **Communication**
 The protocol requires a communication between the actors. Some messages must be sent secretly P2P, whereas the rest are "broadcasted", i.e. sent to all the actors. In either case, we assume there's the underlying mechanism for this. In particular we require the following:
 - All the messages are protected against tampering
 - Impersonation is not feasible, each message's sender can be verified
 - P2P message contents are secret, i.e. only the recipient should be able to read it in plaintext. The messaging metadata (the even fact of communication, message size, etc.) is not secret.

**Scalar derivation**
We'll need a collision-resistant function to generate a scalar from arbitrary public arguments. Basically it's implemented via a hash function (such as SHA-256). However additional steps are required to make sure the result is a valid EC scalar (clamping).
We'll call this function $Hash_s(msg)$

**Mutual antisymmetric secret**

For each pair of actors $i \neq j$ we'll define this function:

$$
\delta(i,j,context) = Hash_s( DH(i,j) | context) \cdot (x_i - x_j)
$$

whereas:
- $DH(i,j)$ stands for Diffie-Hellman shared secret (in terms of their SlatepackAddress)
- $context$ is arbitrary message
- $x_i$ is derived from the actor's address (more about this later)

Note that this function is antisymmetric: $\delta(i,j) + \delta(j,i) = 0$


### Wallet initialization

In this process, N users initialize the wallet

(1) N initial actors decide to initialize/restore the wallet. They agree on M: the minimal quorum needed to build future transactions.

Each actor creates M pseudo-random initial coefficients (perhaps derived from its seed phrase). This information is broadcasted:
- Its address: $addr_i$
- M Commitments to the coefficients: $C_{i,m} = G \cdot\ r_{i,m}$
- PoP (proof of possession) to the above commitments. Schnorr's signatures of its address commitment, signed by its secret coefficients $r_{i,m}$

(2) Each actor receives and verifies this information:
- All addresses must be distinct
- All Commitments must be valid (valid EC points), and appropriate PoP are valid as well

The SSS polynomial is defined as the sum of the partial polynomials provided by N actors:

$$
sk(x) = \sum_{m=0}^M x^m \cdot \sum_{i=1}^N r_{i,m} = \sum_{m=0}^M x^m \cdot s_m
$$

or in terms of pubkeys:

$$
P(x) = \sum_{m=0}^M x^m \cdot \sum_{i=1}^N C_{i,m} = \sum_{m=0}^M x^m \cdot S_m
$$

Note that the coefficients { $S_m$ } and the polynomial $P(x)$ are known, whereas { $s_m$ } are secret, and never computed in a plain form.

At this point the x-coordinate for each validator is computed:

$$x_i = Hash_s("actor-" | addr_i)$$

All the { $x_i$ } must be distinct. Since the addresses of the actors are distinct, we assume the probability of collision of $x_i$ can be neglected.

Each actor computes ands sends **privately** the following partial shares to other validators:

$$
sk_{i,j} = \sum_{m=0}^M r_{i,m} \cdot x_j^m
$$

(3) Each actor that receives its designated partial shares finally calculates its share:

$$
sk_j = \sum_{i=1}^N sk_{i,j} = \sum_{i=1}^N \sum_{m=0}^M r_{i,m} \cdot x_j^m = sk(x_j)
$$

Each actor verifies that its share is correct:

$$
G \cdot sk_j = P(x_j)
$$

Finally the secret polynomial is redefined in terms of the secret shares. For each argument $x$ the polynomial is evaluated by a quorum Q of M actors as:

$$
sk(x) = \sum_{j \in Q} sk_j \cdot \prod_{i \in Q, i \neq j} \frac{x - x_i}{x_j - x_i} = \sum_j sk_{j,Q}(x)
$$

The terms $sk_{j,Q}(x)$ will be called **partial keys**.

Once the ceremony is complete, each actor saves this information:
- Public polynomial coefficients: { $S_m$ }
- Its address: $addr_j$
- Its share: $sk_j$



(4) Coin blinding factor is defined according to this formula:

$$
x_{coin}(number, value) = Hash_s("coin-" | number | value)
$$

and then:

$$
sk_{coin}(number, value) = sk(x_{coin}) + HKDF(S_0, x_{coin})
$$

whereas $number$ specifies the coin ID (number or any unique parameters), and ${S_0}$ is used as a seed to derive a complemental key via HKDF. While this term is known among actors, it's important to keep it, since it can increase the overall security (more about this later).

Note that the coin blinding factor can't be calculated by individual actors (since the function $sk(x)$ is unknown). Actors can only calculate their shares to the coin blinding factor.
But the public key, and the coin commitment can be calculated be each actor individually.


### Adding an actor

A quorum Q of M current actors can add an additional actor. For this the following steps are performed.

(1) New actor picks and shares its unique address. The address must be unique (no duplicates with existing actors)

(2) Each actor computes and sends **privately** the following key share:

$$
sk_{share_i} = sk_{i,Q}(x_{new}) + \sum_{j \in Q, j \neq i} \delta(i,j, context)
$$

whereas $context$ includes all the parameters of the ceremony: $S_0$, $x_{new}$, and the selected quorum Q. This is to make sure $\delta(i,j, context)$ is unique in each ceremony

The new actor then performs the summation of all the shares:

$$
sk_{new} = \sum_{i} sk_{share_i}
$$

Note that during this summation, all the $\delta(i,j)$ terms are cancelled-out. They are needed to blind each actor's secret key, but don't affect the sum of all the shares.

Finally the new actor verifies the correctness of its secret key, by checking if its appropriate pubkey sits on the same SSS polynomial.

$$
G \cdot sk_{new} = P(x_{new})
$$

### Wallet restoration

If some actors lost access to their data, whereas at least M other actors retain their wallets - they can re-evaluate the shares of those actors by the procedure described above: "Adding an actor".
Otherwise, if there're fewer validators remaining, the wallet initialization procedure can be repeated from scratch. Since the whole procedure is deterministic, then the same set of N actors will yield the same secret polynomial and the same shares will be computed.
Otherwise, if there're less than M valid actors remain, and it's not possible to get the initial N actors to repeat the initialization ceremony - it won't be possible to restore the wallet.


### Building a transaction

MW transaction consists of inputs, outputs at least one kernel. In order to build a valid transaction, a quorum of M actors is selected. Each actor gets the transaction parameters:
- list of input coins (coin number + value)
- list of output coins (coin number + value)
- any additional metadata (current blockchain height, fee, the identity of the tx peer, memo, etc.)

Each actor receives and realizes the tx parameters. In particular, the tx balance (outputs minus inputs, the net value received/sent) is calculated. And then it decides if it wishes to take a part. Then the selected actors perform an MPC to build the transaction.

### Inputs

Inputs are represented by Pedersen commitments (or, alternatively, Switch commitments). Since they are EC points, they can be calculated by each actor:

$$
C(coin) = P(x_{coin}) + G \cdot HKDF(S_0, x_{coin}) + H \cdot value
$$

#### Outputs

The UTXO is represented by its commitment and the Rangeproof (a.k.a. bulletproof). The commitment is computed exactly as for inputs, by each actor individually. So, only the rangeproof requires MPC.

During the rangeproof creation, various pseudo-random EC scalars are generated. There're 2 sources of the randomness:
1. Randomness to obscure the UTXO value, also used to "color" it, make it possible to recognize by the owner, and recover the parameters (coin number and the value).
2. Randomness to protect the UTXO blinding factor.

In standard wallet (not multisig) it's possible to use a single source of randomness for both purposes. It also means that UTXO recognition is in fact its full reverse-engineering.

In multisigned wallet it's essential to use 2 distinct sources of randomness. The (1) is derived from the $S_0$ (which is known to all the actors) and the UTXO commitment. The (2) is derived by each actor individually to obscure its key share.
Since it's a multisig, each actor will rely on non-deterministic nonces (random).

The rangeproof protocol between (P)rover and (V)erifier is defined like this:
- P: Commitment, A, S (commitments)
- V: y, z (challenges)
- P: T1, T2 (commitments)
- V: x (challenge)
- P: $Tau_X$ (scalar)
- (continued)

The $Tau_X$ is a linear combination of the UTXO blinding factor, and 2 nonces, whereas EC points T1 and T2 are commitments to those nonces. So, the modified algorithm to create the rangeproof looks like this:

- Each actor proceeds according to the scheme, up to where T1,T2 should be revealed.
- Each actor generates 2 nonces, using true random (i.e. non-deterministic), uses them to calculate its share of T1,T2, and reveals its shares.
- Once M actors reveal their shares, they're aggregated to compute the final T1,T2.
- Each actor derives the next challenge, and uses it to compute and reveal its share of $Tau_X$
- Once M actors reveal their shares, they're aggregated to compute the  $Tau_X$
  - Note: it's possible to verify the correctness of each partial contribution to $Tau_X$
- All the consequent steps are performed individually by each actor (no more MPC is required)


#### Transaction Kernel

In order to create and sign the transaction kernel, the MPC is required as well. Even in normal transactions, where both the sender and the receiver are standard wallets, the MPC is used to create and sign the transaction kernel.

Here we'll generalize this to the case where either side of the transaction (either the sender or the receiver or both) is a multisig wallet.

The difference of the inputs-outputs blinding factors contributes to the transaction excess, and should be compensated by the transaction kernel. Each actor calculates its share of this excess,
this is its share of the secret key that's used to sign the kernel.

An important addition to the classical MW is the so-called transaction **offset**. The total transaction blinding factor excess is split into 2 parts. One is compensated by the transaction kernel, and the other part is revealed in a plain form (EC scalar). This makes it infeasible for the attacker to reverse engineer the coinjoined transactions. To support this, the offset is generated in a deterministic way: $offset = HKDF(S_0, context)$ whereas $context$ accounts for all the transaction parameters: list of coins, metadata, selected quorum, etc.
Then it's accounted in the kernel commitment, and in the signature (during the aggregation stage).

Then the principle is similar to that of UTXO. During the first round each actor creates a non-deterministic nonce, and reveals its image. Then, once all the nonces and kernel commitment shares are known and aggregated (from both sides of the transaction), the kernel is fully built. During the second round, each actor derives the signature challenge, and reveals its share of the blinded secret key.

#### E2E flow

Here we'll consider an example where Alice sends funds to Bob, both are in fact multisig wallets

- Alice side
  - A quorum of $M_A$ actors that co-own the Alice wallet decides to send funds to Bob. A list of input coins is selected, fee is decided, the coin number for the exchange coin is choosen.
  - The information is distributed among the actors.
  - They generate appropriate nonces (2 for output UTXO, one for tx kernel)
  - Round 1: they reveal the images: T1,T2 for the UTXO, and the kernel nonce image.
  - The semi-built kernel is sent to the Bob, among with the general transaction info
- Bob side
  - A quorum of $M_B$ Bob's actors decides to accept the funds. They choose the coin number for the output UTXO
  - Each actor generates the appropriate nonces (2 for output UTXO, and 1 for kernel).
  - Round 1: they reveal the images: T1,T2 for the UTXO, and the kernel nonce image.
  - Round 2: each actor receives the aggregates for the UTXO and the kernel. And reveals its partial signatures
  - The aggregated kernel, offset and Bob's UTXO is sent to Alice
- Alice side
  - Round 2: each actor receives the aggregates for the UTXO and the kernel. And reveals its partial signatures
- The transaction is aggregated, and can be sent to the network

As can be seen from the above, the roles of Bob and Alice actors are symmetric. Both participate in 2 rounds of MPC.

## Security considerations

### Adding the new actor, and the role of $\delta(i,j)$

During the new actor addition, each current actor reveals its secret key in a plain form to the new actor. It's multiplied by the SSS coefficient, but this coefficient is widely known. So without the addition of $\delta(i,j)$ terms each actor secret key would be trivial to compute.

This is why $\delta(i,j)$ terms are essential, they perform the perfect hiding of the actor secret key. Moreover, even if several current actors collude with the new actor, still its key is perfectly hidden as long as there is at least a single honest actor that doesn't disclose its $\delta(i,j)$ term.

The only situation where the actor key can be compromised is where all M-1 other actors collude with the new one. But this essentially means that they together form a quorum of M malicious actors. And obviously a quorum of M actors can calculate and compromise any key.

### Why this initialization procedure

In a previous variant of the proposal, during the initialization M initial actors contributed each a single point to the polynomial. And then, a polynomial of degree M-1 was fully defined by M distinct points. The remaining N-M validators got their shares evaluated later, by the "Add new actor" ceremony.

In the current version, the scheme was changed, it's now equvalent to the standard Feldman DKG scheme. Its advantages are:
- All N actors take part in the initialization, and contribute to the randomness
- It's well-studied, and considered secure

The drawback of this scheme, perhaps minor but should be mentioned, is that actors cannot restore their shares solely from their seed phrase. In the previous share M initial actors computed their shares in advance, solely from their seed phrases. In the current scheme this is not possible, because the polynomial structure and its value at any point is affected by all N actors.

### Address grinding

By grinding we assume that an actor can try different variants of its address "for free", until it gets a desired $x_i$. It's not fesible to reach a collision, whereas its $x_i$ will coincide with other actor's $x_j$, or with $x_{coin}(number, value)$, the argument of a coin blinding factor. However, it's believed that attacker _may_ obtain a desired $x$ to influence the algebraic structure of the Lagrance coefficients ( $\frac{x - x_i}{x_j - x_i}$ ). By such it won't be able to obtain the secret keys, but it can try to cause some correlation between different derived keys (which otherwise should be looking perfectly random).

While there's no direct attacks, and it's more like a paranoid threat, still such things should be avoided nevertheless. If actor addresses are known in advance, then there's no problem. The problem may manifest if the last actor has the ability to observe all other addresses and then grind its own address. This situation should be avoided.

If the latter is the case, then the initialization, and adding new actor schemes should be extended to an additional pre-step. First and foremost, all actors must send a commitment of their address. Only then, after all the commitments are exchanged, all the actors reveal their addresses.

### UTXO and kernel signing, why rely on non-deterministic random

It's generally considered better to rely on deterministic nonce generation scheme, such as RFC-6979. However such schemes can't be used as-is in multisig rituals. There're advanced schemes, such as those used in MuSig-DN, that rely on ZKP to verify that each actor generated its nonce deterministically.

For the sake of simplicity, we'll stick to random (non-deterministic) nonces. In the future it'll be possible to upgrade the protocol, and use the PRF (pseudo-random function) together with Bulletproof ZKP to generate and verify the correctness of the nonces.
In either case, the general flow remains the same.
And as long as the main principle holds: each nonce is used to only answer one challenge - the secret keys are safe.

### Rogue key attack
The only situation where such an attack is possible is during the wallet initialization, where each actor reveals broadcasts the commitments to its coefficients $C_{i,m} = G \cdot\ r_{i,m}$. If not mitigated, an actor can essentially cancel the keys of other actors, and gain an exclusive access.
As we mentioned, this is mitigated by mandatory inclusion of PoP (proof-of-possession) for each commitment.

### Why the coin value should be used together with the coin number in its key derivation

After the agreed transaction is built and signed by the actors, a malicious actor can try to replace transaction inputs (or outputs) by the other ones, containing a lower value. Then, the value excess can be compensated by attacker's UTXO added to the transaction.

Suppose a multisig wallet owns two coins, with values $V1 < V2$. There's a decision to spend the coin with value $V1$ in a transaction. It's collectively built and signed, but then a malicious actor changes the input to $V2$, and appends its UTXO that absorbs the value $V2-V1$.

Actions such as replacing coins are generally not possible, because different coins have different blinding factors, and it's not feasible to build a valid transaction when the blinding factor balance isn't compensated. But what if the attacker can manipulate inputs and outputs such that the **resulting blinding factor balance is unchanged**?


Coins with different numbers (i.e. IDs) will have different blinding factors. But what if a wallet owns several coins with different values but the same number?
Normally actors should not take part in UTXO creation with coin number that was already used. But the situation may be confusing because of the blockchain state volatility. There can be potential reorgs, whereas transactions may be reverted, and then, after some time, included again in a block.
By such it's theoretically possible the wallet will own several coins with the same ID, but different values. If the same blinding factor is used in all of them, then it's trivial to replace them in an already-built transaction.

**Mitigation:** include the coin value in the blinding factor derivation too.

### Wagner attack, Prouhet-Tarry-Escott (PTE) problem

Due to the nature of MW, a transaction may contain multiple inputs and outputs. And the attacker can try to find different sets of inputs/outputs, such that the excess of blinding factors will be the same:

$$
\sum_{i}^{inputs_1} Key(coin_i) - \sum_{j}^{outputs_1} Key(coin_j) = \sum_{i}^{inputs_2} Key(coin_i) - \sum_{j}^{outputs_2} Key(coin_j)
$$

Note that if the attacker could derive blinding factors for arbitrary coin parameters, then it'd be a feasible task for a large enough set. This is known as the Generalized Birthday problem, or Wagner attack.

However, in our particular case, coin blinding factors are derived from the SSS polynomial, whose coefficients are secret.

So, the only way for the attacker to find such sets is to try to find the sets where different powers of $x^n$ are equal for different sets. That is:

$$
\sum_{i}^{inputs_1} x^n(coin_i) - \sum_{j}^{outputs_1} x^n(coin_j) = \sum_{i}^{inputs_2} x^n(coin_i) - \sum_{j}^{outputs_2} x^n(coin_j)
$$

$$\forall\, n \in \{0, \dots, M-1\}$$


(This is like trying to solve M-1 generalized birthday problems simultaneously).

It's a known problem, called Prouhet-Tarry-Escott (PTE) problem. And it's considered generally infeasible to solve, especially for large M.

So, the first mitigation would be: M must be high enough (TBD). If the signature threshold (M/N) needs to be low, then the initialization scheme can be generalized in this way:
- Use higher polynomial degree
- Generate more shares
- Give each actor more shares

In simple words: make each actor behave as several ones, by such artificially raise the number of shares and the degree of the polynomial.

Additional steps can be taken by the actors to prevent the possibility of such attacks (though not sure if they're really needed). Those include the following:
- Ensure the being-spent coins actually exist and unspent (i.e. don't build transaction for non-existing inputs)
- Stick to consequent coin number allocation. Each new coin number should not be completely random.
- Don't create excessive number of outputs in a transaction. Normal transaction creates only a single output (per each side).

### Why it's beneficial to include the commonly-known HKDF-based key complement

Theoretically coin blinding factors should never be revealed. However, if a coin blinding factor is leaked (for whatever reason), without the HKDF-based blinding factor it may compromise all the secret keys.

Without the HKDF part, the coin blinding factor is:

$$
sk(x) = \sum_{n=0}^{M-1} s_n \cdot x^n
$$

whereas $s_n$ are the unknown coefficient. If the coin blinding factor and its number is leaked, then this gives an equation. If M different blinding factors are leaked - then there're M linear equations for $s_n$ coefficients, that can easily be solved.

So, keeping this in mind, it's beneficial to add additional "shared" blinding factor to each key.
