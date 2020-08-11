
- Title: `wallet-(re)play-protection`
- Authors: Phyro, John Tromp, Yeastplume
- Start date: Aug 9, 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

A few months ago, a new class of attacks was found on Grin that we call (re)play attacks. A Mimblewimble transaction that already happened can be replayed if the exact conditions are recreated. In order to be able to replay a transaction the following conditions must be true:
1. The inputs must be in the utxo set
2. The outputs of the transaction must not be in the utxo set (Grin does not allow duplicate outputs in the utxo set)

This means that if Alice sent some coins to Bob and the outputs have been spent, anyone that saw their original transaction could replay the transaction if the same inputs existed in the utxo set. But why would the same inputs exist on the chain in the first place? While this seems harmless at first, it can with some creativity and careful coordination be used take someone else's coins without their permission. The attack is not easy to pull off, but is doable in some scenarios. Similarly, a play attack comes from the same reasoning, but with the difference that a transaction never made it to the chain for some reason. One such reason could be that an input was spent before it was broadcasted which would make the current transaction invalid. In this document, we propose new wallet behaviours that make it robust in the face of (re)play attacks so the end users can't be victims of these attacks.

# Motivation
[motivation]: #motivation

The goal of this RFC is to propose new wallet rules that, when strictly followed, protect the user from all known malicious (re)play attacks. This is done by changing the transaction building process but at the same time keeping it configurable enough to still allow the user to take control of their privacy. The solution mostly supports all the transaction building flows that were used before while encouraging default use of PayJoin transaction with some probability. Contributing inputs on to a 'receive' transaction can be turned off in the wallet configuration settings while still keeping the protection against replay attacks. The wallet also allows choosing different wallet configurations for different use cases e.g. a withdrawal from an exchange never does a PayJoin transaction. 

# Community-level explanation
[community-level-explanation]: #community-level-explanation

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

The key idea here is to observe that `T2` can't be replayed by anyone else. Only we can create the input needed for the transaction to be replayed `O2`. The attacker could attempt to replay the previous transaction `T` to create the  `O2` input, but this would only be possible if they managed to replay transaction `T` which can't be replayed because it has an `anchor` output that has not been spent and hence can't be valid. This means that our transaction `T2` is protected against other parties replaying it. 

If we again used `O3` as an input in a new transaction `T3` to send money to Charlie, the same protection would be put in place. The attacker would need to replay `T2` to recreate `O3` input, but we have already shown that `T2` can't be replayed.

```
O3  -> O4 (change output)
    -> Charlie_output
```

This is the main idea used in this proposal to protect against replay attacks. This is why we name the outputs we created along side the `anchor` output `Protected` outputs. They provide protection for subsequent transactions we make if we use them as inputs to our transactions. Key thing to note here is that while we started with 2 protected outputs `O1` and `O2`, the outputs that we create in a transaction to which we add a protected input automatically become `Protected` outputs as well. This property is what allows us to continue creating new protected outputs without the need to create more than one `anchor` output.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Wallet rules for replay protection

Most common transaction types are:
1. A 1-2 `Regular` tx - 1 input, 2 outputs
2. A 2-2 `PayJoin` tx - 2 inputs (one from each party), 2 outputs

Let's call outputs that hang from a chain that leads to your anchor a `Protected` output. A transaction is protected from replays if a party contributes a Protected output as an input in the transaction. It's not enough that some other party contributes such an input. We are only really protected from replay attacks if we contribute the `Protected` output. Otherwise, our security depends on the honesty and following of the wallet rules of other participants which should not be relied on.

Let's say that we want to be able to do both `Regular` and `PayJoin` transactions. We do both of these so we have some protected outputs and some that are not protected. The first problem we have is that after a wallet restore, we can't really tell whether an output is protected through a PayJoins+Anchor or not. It would be nice if we could label our outputs as either `Protected` or `Unprotected` after a wallet restore.

### Separation of replay protected outputs

Let's define a clear separation of outputs that are protected from those that are not. We define a protected output generator `GenP` that allows us to either `create` a protected output or `check` whether an output is `Protected`.

An output is labeled as `Protected` *only if* it was created in a transaction where we contributed an input that was labeled as `Protected`. An exception to the rule are outputs that are created as a part of the _Bootstrap_ transaction discussed in the next section.

#### Possible protected output generators implementations

As we mentioned above, `GenP` for `Protected` outputs would need to have `create` and `check` functions defined.

There are a few choices how to label outputs as `Protected` while still being able to identify them across difference devices:
1. Call to `create` uses a new derivation path `P` that is used only for creating `Protected` outputs. Similarly `check` uses the same `P` to check whether an output is protected.
2. Call to `create` creates a specific `r` value for `Protected` outputs e.g. they should start with `N` zeros or be divisible by some number `M`
3. Call to `create` uses additional output information in the ~30 bytes that are available in the Bulletproofs to convey the idea whether the output is protected. Perhaps we could define a specific structure for these bytes e.g. `<scheme_version:1 byte><meta_data:4 bytes><data:25 bytes>` where `metadata` would also tell whether the `data` that follows is encrypted or not. The data could be encrypted using the `seed` key and could thus hold information on output labels which would only be available to the owner of the output. We could even include the starting bytes of the anchor that protects it if we wanted to. If we went such path, we would need to think of the possible drawbacks.

In all cases, the output label should only be visible to the owner of the output. It seems necessary to have labeling information about the UTXO on the UTXO itself if we want it to be consistent with different wallet reusing the same seed. If the information is held only on the wallet side, then we hit much bigger issues because there comes a need for either a manual intervention and labeling or a robust solution for synchronization between wallets - which does not appear simple to build.

### Simple bootstrapping of protected outputs

We start off with all of our outputs marked as `Unprotected`. To create a `Protected` output out of nothing we create the following outputs:
1. an `anchor` output that has a form `0*H + r*G` - generated from key derivation path `A`
2. a set of protected outputs `PS` that hold our coins - generated from key derivation path `P`

We make a bootstrap transaction with `anchor` and `PS` outputs. The outputs in `PS` are exceptionally labeled as `Protected` because they are joined in the same transaction output set as our anchor output. This transactions cannot be replayed because the anchor output will never be spent which means that `PS` outputs are safe from being recreated.

_Note: Grin has a rule that an output that already exists in the UTXO set cannot be created which makes the transaction invalid._

### Wallet transaction rules

There are two known ways to protect against replay attacks using wallet rules. One is to utilize the `Protected` outputs in transactions, the other one is to keep a history of outputs that were spent and label as 'unsafe' those that the wallet does not remember creating.

#### Replay protection with utilization of Protected outputs

In order to protect ourselves from replay attacks, we need to follow a simple rule:
**When we are sending money to someone , we _MUST_ always include a `Protected` input as a part of the transaction.** This is to prevent someone doing a replay of the transaction which would move money from our outputs. Exception to this rule are self-spends.

This means that:
1. We can make regular 1-2 transaction if our input is labeled as `Protected`
2. We can receive money through a 1-2 transaction

The only thing we need to be aware is that our output that is created in a 1-2 receive transaction will be labeled as unprotected and will hence need to be spent in a transaction that will include one of our `Protected` outputs as an input. A more general rule is that outputs created in a transaction to which we did not contribute a `Protected` input are unprotected and need to be spent along side some of our `Protected` inputs. In theory, it should be impossible to create replay any transaction if this rule was followed.

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
  if anchor_not_exists:
    return [], bootstrap_outputs()
  // PayJoin - protect the transaction with a protect output as an input
  return [random_protected_output()], []
}

// Receive txs don't need an absolute protection so we flip a coin to determine whether
// it will be a regular or protected transaction based on the wallet configuration
fn receive(value: int64) -> ([]Output, []Output) {
  receiver_inputs = []
  receiver_outputs = []
  // receive-only wallets could have receive_protected_prob=0.0 to always have 1-2 receive transactions
  is_protected = random() <= config.RECEIVE_PROTECTED_PROB
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

// TODO: needs handling of running out of protected outputs and keeping a minimal of `N`
// protected outputs as defined in the configuration file
```

TODO: Check this is safe.

#### Protection with wallet history

A wallet can remember which outputs were spent. This way, if a spent output reappears in the wallet, a user is given a choice to either accept it or refresh it through a self-spend transaction.

The downside of this approach is that a replay attacks are still possible in which case it would mean that a user would need to make a manual choice what to do with outputs that are unknown to the history of the wallet. Unknown outputs are also those created from the same seed on a different device and from child wallets since they each have their own history. The cross device scenario could be mitigated by export and import of wallet history and the child wallet outputs can be labeled and hence assumed as safe. It's up to the child wallet to protect itself from the attacks.

### Receive-only wallets
 
Any kind of automated receiving should default to 1-2 transactions and thus creating _unprotected_ outputs to avoid utxo spoofing attack which would reveal our inputs. Always performing 1-2 receive transactions can be achieved by setting the configuration `RECEIVE_PROTECTED_PROB = 0.0` which is explained in the next section.

### Transaction building configuration

```yaml
// Defines a list of wallet configurations to define different behaviour that we do as a receiver of a
// transactions. All wallet configurations inherit the values from the 'default' option that can be
// overriden. A wallet should allow the user to generate a one-time SlatepackAddress based on a chosen
// configuration. 
TRANSACTION_BUILDING_CONFIGURATIONS:
  default:
    // Probability that we will protect a 'receive' transaction from replay attacks. If an anchor does not
    // exist, the transaction will be protected by generating an anchor and an initial set of protected outputs.
    // If such an anchor already exists, we contribute a protected output as an input which makes it a PayJoin
    // transaction. By default, half of the receive transaction will be PayJoin transactions. This is to help mask
    // the sender inputs. Additionally, PayJoin transactions are cheaper and more healthy for the network because
    // they don't generate new outputs.
    // If PayJoin transactions are not wanted due to privacy concerns, you can set this value to 0.0 in which case
    // receive transactions will never contribute an input.
    // Default: RECEIVE_PROTECTED_PROB = 0.5
    RECEIVE_PROTECTED_PROB: 0.5

    // The number of protected outputs we want to have available at any time. This is to be able to have concurrent
    // transactions that are safe from replay attacks. If we had only 1 protected output and wanted to make two
    // 'send' transactions, we would need to do them sequentially because we could only protect a single
    // transaction at once. This defines the minimal set size of protected outputs. Exchanges would want to set
    // this to higher values to avoid blocking transactions.
    N_PROTECTED_OUTPUTS: 5

  // PayJoins configuration
  all_payjoins:
    RECEIVE_PROTECTED_PROB: 1.0

  // No PayJoins configuration - e.g. could be used for withdrawals from exchanges
  no_payjoins:
    RECEIVE_PROTECTED_PROB: 0.0
    
// We can predefine some fixed transaction building configurations for specific addresses, but we should note
// that this comes at a privacy risk due to the SlatepackAddress reuse.
ADDRESS_TRANSACTION_CONFIGURATIONS:
  grin1dhvv9mvarqrqwerfderuxp3qgl6qpphvc9p4u24asdfec0mvvg6342q4w6: default
  grin1hhb34mfsrqwlfderuxpfdql6qpphvc9p4u24fd47ec0mvvg6342q1asdxc: all_payjoins
  grin1fsdf9fd83e3fjvcxruxp3qgl6qpphvc9p4u24347ec0mvvg6342fdoiosl: no_payjoins
```

TODO: Automated wallet `receive` actions open up the receiver to dusting attacks if an address can be reused. Should we clearly separate automated `receive` wallets from others? At the very least, `receive` PayJoin transactions should be confirmed manually otherwise the utxos are vulnerable to UTXO spoofing attack. Perhaps we should have `RECEIVE_PROTECTED_PROB: 0.0` by default to avoid leaking inputs? should all transactions by default be inherently interactive and thus requiring a manual step from the receiver?

### Child wallets

TODO

### Exchanges scenarios

#### Exchange configuration

As mentioned in the configuration section, exchanges would likely need a bit bigger set of protected outputs so they would need to adjust the `N_PROTECTED_OUTPUTS` configuration. It would be encouraged that exchanges have `RECEIVE_PROTECTED_PROB` set to `1.0` to help mask the user input.

#### User withdrawing from an exchange configuration

We probably don't want to be doing a PayJoin transaction when we are withdrawing from an exchange to showing them one of our inputs. To avoid this, we can generate a `SlatepackAddress` from a configuration option that has 0% chance to do a PayJoin receive transaction.

### PayJoin transactions

PayJoin transactions when mixed with regular transaction introduce some uncertainty to the inputs side of a transaction. When Alice is sending coins to Bob, Bob can decide to contribute his own input to a transaction which makes it harder for chain analysis tools to determine who is the owner of an input - sender's inputs become mixed with the receiver's inputs. So it contributes to the sender's privacy, but hurts a bit the receiver's privacy because they show one of their inputs. The only party that knows the receiver contributed an input is the sender though.

Currently, PayJoin transactions are cheaper than regular transactions because they have more inputs. This also means that there are less outputs created which helps to reduce unnecessary chain growth. A 2-2 PayJoin transaction does not create a new output as opposed to a regular 1-2 transaction, which means that it can be thought of as an replacement of the value of the current output with a new value which does not increase the UTXO set size. Another side effect of PayJoin transactions is that backwards chain analysis becomes much harder to follow than it is right now.

### Play attack protection

The difference between a Play and Replay attacks are that in a Play attack, the transaction never lands on the chain. This allows some funny scenarios where Alice wants to pay Bob, but after constructing the transaction and broadcasting it, the transaction does not land on the chain for some reason. Let's say that Alice and Bob recreate a new transaction and Alice uses different outputs. This second transaction goes through and both Alice and Bob are happy. However, if Bob saw the first transaction, if it becomes valid at some point, it is possible for him to 'play' it or broadcast it on the chain after the second transaction was already done. This way, Bob receives his payment twice. This means that a Play attack attacks the sender by tricking them into signing a transaction multiple times. Transactions that don't make it to the chain should _always_ be cancelled by the user before sending a new transaction.

To protect against this, a user should have an option to cancel a transaction which would label the sender's inputs as `Must use` in the next transaction. The reason why the sender would want to reuse the inputs is to make the previous transaction invalid. So now, only 1 of the two transactions that have been signed can make it to the chain. If the transaction failed to get on the blockchain again and Bob 'gave up', it should be cancelled and an immediate self-spend should be done to prevent giving Bob the possibility of playing the transaction after.

# Drawbacks
[drawbacks]: #drawbacks

It requires an `anchor` input that is never spent which increases the chain 700 bytes per wallet. These outputs might be easier to identify because they never move. How easy/hard would it be to identify them is unclear because each wallet is expected to have only one such output and a lot of wallets will get lost and hence a lot of outputs will never move.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Seems to be able to protect against all replay attacks if we follow a simple rule which means it doesn't require any consensus changes. There are also some other ideas on how to tackle this problem that solve the problem at the consensus level. Each implementation has its own set of tradeoffs. The main benefit of non-consensus solutions is that they can be replaced later with any alternative, including a consensus solution.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What should the transaction building configuration look like?
- Should we label `Protected` outputs by making them have a separate derivation path?
- Is it worth implementing other solutions as well (e.g. out
- put history + sweeping) which seem to have more problems and introduce complexity to wallet handling?
- Should the default probability for a PayJoin on a 'receive' transaction be 0.0 to avoid leaking outputs?
- Should certain transaction options allow having a user-level interactive transaction which means that the user must manually confirm the transaction building process? Perhaps have this as an additional transaction building configuration which would also allow selecting specific outputs to be used in a transaction?

# Future possibilities
[future-possibilities]: #future-possibilities

The options of transaction building configuration and programmability are only limited by our imagination. PayJoins could be researched a bit more and used more often if we don't interact with parties that are known to collect data on their users.

# References
[references]: #references

[Replay attacks and possible mitigations](https://forum.grin.mw/t/replay-attacks-and-possible-mitigations/7415)

[PayJoins for replay protection](https://forum.grin.mw/t/payjoins-for-replay-protection/7544)