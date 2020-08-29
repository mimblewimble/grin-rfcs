
- Title: `wallet-(re)play-protection`
- Authors: Phyro, John Tromp, Yeastplume
- Start date: Aug 9, 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

Given certain conditions, UTXOs that belong to a user can be moved without their explicit permission. This document outlines the fixes that prevents this behaviour from occuring by introducing additional rules a wallet should follow.

# Motivation
[motivation]: #motivation

A new class of attacks was found on Grin that we call (re)play attacks. A Mimblewimble transaction that already happened can be replayed if the exact conditions are recreated. In order to be able to replay a transaction the following conditions must be true:
1. The inputs must be in the utxo set
2. The outputs of the transaction must not be in the utxo set (Grin does not allow duplicate outputs in the utxo set)

This means that if Alice sent some coins to Bob and the outputs have been spent, anyone that saw their original transaction could replay the transaction if the same inputs existed in the utxo set. But why would the same inputs exist on the chain in the first place? Alice could send those coins to Bob again, using different inputs but the same output for Bob, and trick Bob into thinking that this constitutes a new payment. If Bob accepts it as such, then Alice can replay Bob's spend of the recreated output at an opportune moment. Similarly, a _play_ attack comes from the same reasoning, but with the difference that a transaction never made it to the chain for some reason. One such reason could be that an input was spent before it was broadcasted which would make the current transaction invalid. In this document, we propose new wallet behaviours that make it robust in the face of (re)play attacks so the end users can't be victims of these attacks.


# Community-level explanation
[community-level-explanation]: #community-level-explanation

We propose that wallets follow the following 2 rules to fully protect users from (re)play attacks.

- Spend safely
- Cancel safely

An `anchor` is a 0-valued output (created by this wallet) that is never spent.
A safe (from this wallet's viewpoint) transaction is a transaction that either creates an anchor, or that spends an output from an earlier safe transaction.
Unsafe receives are allowed, but safe receives (that are generally payjoins) are preferred as long as the receiver gets to finalize, which minimizes the risk of utxo spoofing. Sends should always be a safe transaction.

A safe cancel requires an immediate self-spend of an input of the tx to be canceled.

The wallet rules for spending safely would happen behind the scenes, so the end user would not be aware of any changes. Canceling a transaction would be a different experience than it is now, but it is required to change it to prevent the play attacks from occuring. A possible cancel flow would be to make the cancellation a longer event which is not finished until the self-spend transaction has been confirmed on the chain.  

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## General technical overview

As we have already mentioned, a transaction can be replayed if the inputs are recreated and its outputs don't exist. One way to stop this attack would be to make it impossible to recreate an input. Of course the owner can always recreate it, but the problem is really if anyone else recreates it by replaying the transaction that created the input. Luckily, there is a way to make an input that is protected and can't be recreated by anyone else except by the owner. This seems impossible because you can always just go one transaction back and recreate the outputs that created the inputs and keep repeating this until you hit a transaction that can be replay at which point you can replay all the next transactions and thus successfully recreate the target input. To prevent this, we create a 'stopping' transaction `T` for which we _know_ it can't be recreated. We achieve this by creating an output in `T` that we call an `anchor` output which will never be spent. Since Grin does not allow duplicate outputs, it means that the transaction that created `anchor` output can't be recreated because it would attempt to insert a duplicate output in the UTXO set which is invalid by Grin rules. Along with an `anchor` output we create additional outputs `O1` and `O2` in `T` which we call `Protected` outputs - it's not obvious why they have this name yet.

Let's say we have a transaction `T`
```
I1  -> anchor
    -> O1
    -> O2
```

We now use `O2` in a new transaction `T2` as an input to send money to Bob.

```
O2  -> O3 (change output)
    -> Bob_output
```

The key idea here is to observe that `T2` can't be replayed by anyone else. Only we can create the input `O2` needed for the transaction to be replayed. The attacker could attempt to replay the previous transaction `T` to create the  `O2` input, but `T` can't be replayed because it has an `anchor` output that has not been spent and hence can't be valid. This means that our transaction `T2` is protected against other parties replaying it. 

If we again used `O3` as an input in a new transaction `T3` to send money to Charlie, the same protection would be put in place. The attacker would need to replay `T2` to recreate `O3` input, but we have already shown that `T2` can't be replayed.

```
O3  -> O4 (change output)
    -> Charlie_output
```

This is the main idea used in this proposal to protect against replay attacks. This is why we name the outputs we created along side the `anchor` output `Protected` outputs. They provide protection for subsequent transactions we make if we use them as inputs to our transactions. Key thing to note here is that while we started with 2 protected outputs `O1` and `O2`, the outputs that we create in a transaction to which we add a protected input automatically become `Protected` outputs as well. This property is what allows us to continue creating new protected outputs without the need to create more than one `anchor` output.


## Wallet rules for replay protection

Most common transaction types are:
1. A `Regular` tx which has inputs from a single party - the sender
2. A `PayJoin` tx which has inputs from both parties - both sender and receiver contribute at least one input each

Let's call outputs that hang from a chain that leads to your anchor a `Protected` output. A transaction is protected from replays if a party contributes a Protected output as an input in the transaction. It's not enough that some other party contributes such an input. We are only really protected from replay attacks if we contribute the `Protected` output. Otherwise, our security depends on the honesty and following of the wallet rules of other participants which should not be relied on.

Let's say that we want to be able to do both `Regular` and `PayJoin` transactions. We do both of these so we have some protected outputs and some that are not protected. The first problem we have is that after a wallet restore, we can't really tell whether an output is protected through a PayJoins+Anchor or not. It would be nice if we could label our outputs as either `Protected` or `Unprotected` after a wallet restore.

### Separation of replay protected outputs

Let's define a clear separation of outputs that are protected from those that are not. We define a protected output generator `GenP` that allows us to either `create` a protected output or `check` whether an output is `Protected`.

An output is labeled as `Protected` if it was created in a transaction where we either contributed a protected input, or contributed an anchor output.

#### Possible protected output generators implementations

As we mentioned above, `GenP` for `Protected` outputs would need to have `create` and `check` functions defined.

There are a few choices how to label outputs as `Protected` while still being able to identify them across difference devices:
1. Call to `create` uses a new derivation path `P` that is used only for creating `Protected` outputs. Similarly `check` uses the same `P` to check whether an output is protected. Perhaps we could have `N` labels possible which would be labeled by the `r % N` result
2. Call to `create` uses additional output information in the bytes that are available in the Bulletproofs to convey the idea whether the output is protected. These bytes are right now 'zero' bytes meaning their binary representation is all zeros. We label an output as `Protected` by setting the `label` bit that is unused now to `1`. `Unprotected` outputs will have the label bit set to `0` which includes all the old outputs as well. The label bit can be read only by the owner of the output because the message that is encoded in the Bulletproof is already private. If we went such path, we would need to think of the possible drawbacks.

In all cases, the output label should only be visible to the owner of the output. It seems necessary to have labeling information about the UTXO on the UTXO itself if we want it to be consistent with different wallet reusing the same seed. If the information is held only on the wallet side, then we hit much bigger issues because there comes a need for either a manual intervention and labeling or a robust solution for synchronization between wallets - which does not appear simple to build.

### Simple bootstrapping of protected outputs

We start off with all of our outputs marked as `Unprotected`. To create a `Protected` output out of nothing we create the following outputs:
1. an `anchor` output that has a form `0*H + r*G` - generated from key derivation path `A`
2. a set of protected outputs `PS` that hold our coins - generated from `GenP`

We make a bootstrap transaction with `anchor` and `PS` outputs. The outputs in `PS` are exceptionally labeled as `Protected` because they are joined in the same transaction output set as our anchor output. This transactions cannot be replayed because the anchor output will never be spent which means that `PS` outputs are safe from being recreated.

_Note: Grin has a rule that an output that already exists in the UTXO set cannot be created which makes the transaction invalid._

### Wallet transaction rules

There are two known ways to protect against replay attacks using wallet rules. One is to utilize the `Protected` outputs in transactions, the other one is to keep a history of outputs that were spent and label as 'unsafe' those that the wallet does not remember creating.

#### Replay protection with utilization of Protected outputs

In order to protect ourselves from replay attacks, we need to follow a simple rule:
**When we are sending money to someone , we _MUST_ always include a `Protected` input as a part of the transaction.** This is to prevent someone doing a replay of the transaction which would move money from our outputs. Exception to this rule are self-spends that use inputs from an unsafe receive transaction.

This means that:
1. We can make regular 1-2 transaction if our input is labeled as `Protected`
2. We can receive money through a 1-2 transaction

The only thing we need to be aware is that our output that is created in a 1-2 receive transaction will be labeled as unprotected and will hence need to be spent in a transaction that will include one of our `Protected` outputs as an input. A more general rule is that outputs created in a transaction to which we did not contribute a `Protected` input are unprotected and need to be spent along side some of our `Protected` inputs. In theory, it should be impossible to replay a transaction that followed this rule.

_Note: If we are the sender in a 1-2 transaction where we use a `Protected` output as an input, we create a change output which is also a `Protected` output. Since we spent 1 protected input and created another one, it means that the sender can chain such 1-2 transactions safely._

This protects the user from all replay attacks and it can happen behind the scenes without the user knowing there are different types of outputs.

Wallet pseudo-code (WIP):
```rust
// The general idea is to also have at least one protected output available at all times after
// the bootstrap phase. Might want to have a dozen of these in case of high number of transactions
// being performed which would be useful for parties that perform multiple concurrent transactions e.g. exchanges

// Bootstrap transaction outputs by creating an anchor and an initial set of protected outputs PS
fn bootstrap_outputs() -> []Output {
  anchor = generate_anchor_output()
  PS = generate_protected_outputs()
  return [anchor] + PS
}

// A protected version of a receive transaction. Returns a list of inputs and a list of outputs
// In case an anchor does not exist, it contributes an anchor and PS outputs. If an anchor already
// exists, it is a PayJoin transaction - we add a protected output as an input
fn receive_protected() -> ([]Output, []Output) {
  // if we have no anchor, contribute anchor + PS outputs for replay protection
  // TODO: this might be too obvious from the chain analysis perspective
  if anchor_not_exists {
    return [], bootstrap_outputs()
  }
  // PayJoin - protect the transaction with a protect output as an input
  return [random_protected_output()], []
}

// Receive txs don't need an absolute protection so we flip a coin to determine whether
// it will be a regular or protected transaction based on the wallet configuration
fn receive(value: int64) -> ([]Output, []Output) {
  receiver_inputs = []
  receiver_outputs = []
  // receive-only wallets could have receive_protected_prob=0.0 to always have 1-2 receive transactions
  is_protected = random() <= config.receive_protected_prob
  if is_protected {
    receiver_inputs, receiver_outputs = receive_protected()
    receiver_outputs += [generate_protected_output(value)]
  } else {
    receiver_outputs += [generate_unprotected_output(value)]
  }
  return receiver_inputs, receiver_outputs
}

// A 'send' transaction needs to always be protected from replays so we MUST
// either add a protected output as an input or make it a bootstrap transaction
// if an anchor does not exist yet
fn send(value: int64) -> ([]Output, []Output) {
  // first pick the inputs that hold enough value to perform a tx
  inputs = pick_random_inputs()
  outputs = []

  // if we have no anchor, contribute anchor + PS outputs for replay protection
  if anchor_not_exists() {
    outputs = bootstrap_outputs()
  // otherwise, add a protected output as input to protect the transaction
  } else {
    // contribute a protected output as an input if it's not already contributed
    if count_protected_output(inputs) == 0 {
        inputs += [random_protected_output()]
    }
  }
  change_value = sum(input.v for input in inputs) - value  // fee is missing from here
  // we know our change output will be protected either by bootstrap outputs or by
  // a protected input so we label it as a protected output
  change_output = generate_protected_output(change_value)
  outputs += [change_output]
  return inputs, outputs
}

// NOTE: There are many possible strategies that can keep the number of protected outputs
// higher than the specified n_protected_outputs configuration. These can be created by:
// 1. doing self-spend 1-2 transactions (needs to pay fee)
// 2. adding new outputs to existing transactions
// 3. attaching outputs to transactions that pass by and have overpaid the fees
// The handling of these is left to the wallet implementation.
```

TODO: Check this is safe.

#### Protection with wallet history

A wallet can remember which outputs were spent. This way, if a spent output reappears, the default wallet behaviour could be to ignore such output and not count it in the balance. Wallet configuration could allow users to see these outputs and decide to either accept it or refresh it through a self-spend transaction.

The downside of this approach is that a replay attacks are still possible in which case it would mean that a user would need to make a manual choice what to do with outputs that are unknown to the history of the wallet. Unknown outputs are also those created from the same seed on a different device and from child wallets since they each have their own history. The cross device scenario could be mitigated by export and import of wallet history and the child wallet outputs can be labeled and hence assumed as safe. It's up to the child wallet to protect itself from the attacks.

### Receive-only wallets
 
Any kind of automated receiving should default to 1-2 transactions and thus creating _unprotected_ outputs to avoid utxo spoofing attack which would reveal our inputs. Always performing 1-2 receive transactions can be achieved by setting the configuration `receive_protected_prob = 0.0` which is explained in the next section.

### Transaction building configuration

```yaml
// Defines a list of wallet configurations to define different behaviour that we do as a receiver of a
// transactions. All wallet configurations inherit the values from the 'default' option that can be
// overriden. A wallet should allow the user to generate a one-time SlatepackAddress based on a chosen
// configuration. 
transaction_building:
  configs:
    - name: default
      // Probability that we will protect a 'receive' transaction from replay attacks. If an anchor does not
      // exist, the transaction will be protected by generating an anchor and an initial set of protected outputs.
      // If such an anchor already exists, we contribute a protected output as an input which makes it a PayJoin
      // transaction. By default, half of the receive transaction will be PayJoin transactions. This is to help mask
      // the sender inputs. Additionally, PayJoin transactions are cheaper and more healthy for the network because
      // they don't generate new outputs.
      // If PayJoin transactions are not wanted due to privacy concerns, you can set this value to 0.0 in which case
      // receive transactions will never contribute an input.
      // Default: receive_protected_prob: 0.9
      receive_protected_prob: 0.9

      // The number of protected outputs we want to have available at any time. This is to be able to have concurrent
      // transactions that are safe from replay attacks. If we had only 1 protected output and wanted to make two
      // 'send' transactions, we would need to do them sequentially because we could only protect a single
      // transaction at once. This defines the minimal set size of protected outputs. Exchanges would want to set
      // this to higher values to avoid blocking transactions.
      // Default: n_protected_outputs: 5
      n_protected_outputs: 5

    // PayJoins configuration
    - name: all_payjoins
      receive_protected_prob: 1.0

    // No PayJoins configuration - e.g. could be used for withdrawals from exchanges or other untrusted parties
    - name: no_payjoins
      receive_protected_prob: 0.0
    
  // We can predefine some fixed transaction building configurations for specific addresses, but we should note
  // that this comes at a privacy risk due to the SlatepackAddress reuse.
  address_configurations:
    grin1dhvv9mvarqrqwerfderuxp3qgl6qpphvc9p4u24asdfec0mvvg6342q4w6: default
    grin1hhb34mfsrqwlfderuxpfdql6qpphvc9p4u24fd47ec0mvvg6342q1asdxc: all_payjoins
    grin1fsdf9fd83e3fjvcxruxp3qgl6qpphvc9p4u24347ec0mvvg6342fdoiosl: no_payjoins
```

TODO: Automated wallet `receive` actions open up the receiver to dusting attacks if an address can be reused. Should we clearly separate automated `receive` wallets from others? At the very least, `receive` PayJoin transactions should be confirmed manually otherwise the utxos are vulnerable to UTXO spoofing attack. Perhaps we should have `receive_protected_prob: 0.0` by default to avoid leaking inputs? should all transactions by default be inherently interactive and thus requiring a manual step from the receiver?

### Wallet accounts

Each wallet account should have its own `anchor` output protecting it.

### Exchanges scenarios

#### Exchange configuration

As mentioned in the configuration section, exchanges would likely need a bigger set of protected outputs so they would need to adjust the `n_protected_outputs` configuration. It would be encouraged that exchanges have `receive_protected_prob` set to `1.0` to help mask the user input.

#### User withdrawing from an exchange configuration

We probably don't want to be doing a PayJoin transaction when we are withdrawing from an exchange to showing them one of our inputs. To avoid this, we can generate a `SlatepackAddress` from a configuration option that has 0% chance to do a PayJoin receive transaction.

### PayJoin transactions

PayJoin transactions when mixed with regular transaction introduce some uncertainty to the inputs side of a transaction. When Alice is sending coins to Bob, Bob can decide to contribute his own input to a transaction which makes it harder for chain analysis tools to determine who is the owner of an input - sender's inputs become mixed with the receiver's inputs. So it contributes to the sender's privacy, but hurts a bit the receiver's privacy because they show one of their inputs. The only party that knows the receiver contributed an input is the sender though.

Currently, PayJoin transactions are cheaper than regular transactions because they have more inputs. This also means that there are less outputs created which helps to reduce unnecessary chain growth. A 2-2 PayJoin transaction does not create a new output as opposed to a regular 1-2 transaction, which means that it can be thought of as an replacement of the value of the current output with a new value which does not increase the UTXO set size. Another side effect of PayJoin transactions is that backwards chain analysis becomes much harder to follow than it is right now.

_PayJoin transactions also allow for the possibility of the receiver paying their share of fees. Whether users would find this useful is not clear yet._

### Play attack protection

The difference between a Play and Replay attacks are that in a Play attack, the transaction never lands on the chain. This allows some funny scenarios where Alice wants to pay Bob, but after constructing the transaction and broadcasting it, the transaction does not land on the chain for some reason. Let's say that Alice and Bob recreate a new transaction and Alice uses different outputs. This second transaction goes through and both Alice and Bob are happy. However, if Bob saw the first transaction, if it becomes valid at some point, it is possible for him to 'play' it or broadcast it on the chain after the second transaction was already done. This way, Bob receives his payment twice. This means that a Play attack attacks the sender by tricking them into signing a transaction multiple times. Transactions that don't make it to the chain should _always_ be cancelled by the user before sending a new transaction.

To protect against this, a user should have an option to cancel a transaction which would immediately create a self-spend transaction with the same inputs that the canceled transaction used and broadcast it to the network. The reason why the sender would want to reuse the inputs is to make the previous transaction invalid. So now, only 1 of the two transactions that have been signed can make it to the chain. As already mentioned, the cancel would not be confirmed until the self-spend transaction has been confirmed on the chain.

_Note: If we make another transaction to the same user, we should either wait enough confirmations or use the new self-spend outputs that were created and confirmed on the chain as inputs for this new transaction. This is to avoid a possible short reorg attack which could remove the self-spend transaction and publish both transaction which would result in a double-spend._

# Drawbacks
[drawbacks]: #drawbacks

It requires an `anchor` input that is never spent which increases the chain 700 bytes per wallet. These outputs might be easier to identify because they never move. How easy/hard would it be to identify them is unclear because each wallet is expected to have only one such output and a lot of wallets will get lost and hence a lot of outputs will never move.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

There are also other ideas on how to protect against replay attacks that solve the problem at the consensus level. Both wallet and consensus level solutions have their own tradeoffs. The main benefit of solutions at the wallet level is that they can't introduce a consensus failure and because they are only a change in the wallet behaviour, they can later be replaced by a consensus level solution if needed.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Should we use two bits for labeling the outputs to clearly distinguish the old outputs that did not use the labeling scheme? This could come in handy to immediately spend an output that is received with the old scheme as it could be susceptible to a replay attack so it should be immediately spent.
- Is it worth implementing other solutions as well (e.g. output history + sweeping) which seem to have more problems and introduce complexity to wallet handling?
- Should the user have a transaction configuration option that, when enabled, would require a manual confirmation of the receiving transactions? The user manually confirming the outputs they would receive prevents dusting attacks (regardless of the cost) and severely limits any other utxo spoofing methods. It also allows the user to have full control over what outputs they will have in their wallet.

# Future possibilities
[future-possibilities]: #future-possibilities

The options of transaction building configuration and programmability are only limited by our imagination. PayJoins could be researched a bit more and used more often if we don't interact with parties that are known to collect data on their users.

# References
[references]: #references

[Replay attacks and possible mitigations](https://forum.grin.mw/t/replay-attacks-and-possible-mitigations/7415)

[PayJoins for replay protection](https://forum.grin.mw/t/payjoins-for-replay-protection/7544)