- Title: manual-confirmation
- Authors: [David Tavarez](mailto:david@punksec.com)
- Start date: July 14, 2021
- RFC PR: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/)
- Tracking issue: https://github.com/mimblewimble/grin/issues/

---

## Summary
[summary]: #summary

This RFC defines the API changes and additions necessary to give users the ability to manually accept an incoming transaction.

## Motivation
[motivation]: #motivation

Mimblewimble transactions are interactive, this doesn't necessarily mean that parties have to be online, they just need to be able to communicate with each other at some point. This gives the recipient the opportunity to reject an incoming transaction which is currently not possible. The `SlatepackAddress` is not only used to encrypt asynchronous transactions but also to route synchronous transactions. By default when a recipient's wallet receives the request to participate in the transaction building process, the wallet automatically adds the recipient's signature data [1] and returns a slatepack which includes recipient's signature. While automatically adding the recipient's signature data is convenient, it takes away users' freedom to have full control over their own wallets. The ability of refusing incoming transactions is only possible when transactions are interactive, like in Mimblewimble.

Manual acceptance incoming transactions also adds:

- Protection of an Output Injection (or Dust attack) in users' wallet.
- Consistency between Sender-Receiver-Sender (SRS) flow and the Receiver-Sender-Receiver (RSR) one.
- The ability of both parties involved in the transaction to confirm the amount of the transaction.

Manual confirmation helps users understand the interactivity of Grin and are a unique feature that is only possible in protocols with interactive transaction. We also gain the ability for both the sender and the receiver to commit to an arbitrary statement/document with our transaction. This is because they both can read what they're commiting to before they confirm (sign) the transaction.

## Community-level explanation
[community-level-explanation]: #community-level-explanation

This is an opt-in feature, disabled by default, that can be enabled by setting to true a configuration flag named `manual_confirmation`.

When the grin wallet receives a request to build a transaction synchronously, the grin wallet will check for the `manual_confirmation` flag to determine if the manual confirmation feature is on. if it is, the recipient's wallet **will not automatically enter to the transaction creation process with the sender**. The receiver could now accept the incomming transaction manually.

Although this is an optional feature, we should encourage its use.

### Signing the transaction offer

The receiver can then sign the unsigned transaction by specifying the transaction `id` (`-i`) or the `tx-UUID` (`-t`) (obtained by `txs`) to the `sign` command, for example:

`grin-wallet sign -i 4`

After entering the command, the wallet will sign the transaction, generate a slatepack and automatically try to share it with the sender. If the communication with the sender can't be established, the partial signed slatepack will be displayed.

If the flag `-n` `--noshare` is present, the transaction will be signed but the wallet won't try to send it to the sender and will only display the partial signed slatepack. 

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Configuration

### `manual_confirmation`

A new boolean property should be added with the name of `manual_confirmation` in the configuration, this property is optional and the default value is `false`. If the property is not found the value should be set to `false`.

### API changes

### Foreign RPC API

### `receive_tx`

The `receive_tx` endpoint will check the value of `manual_confirmation`, If `manual_confirmation` is `true` the receipient's wallet will:

- Avoid adding the participant data (`excess`, `signature` and `nonce` ).
- Avoid adjusting the transaction offset.
- Avoid adding the output(s) to the transaction.

By doing the above it won't be possible to add the receipient payment proof. The endpoint will then return the same received slatepack.

The status of the transaction in this case will be set as `Receiving (Unsigned)` in the receiver's wallet database.

### API additions

### Foreign RPC API

### `receive_tx_sig`

The `receive_tx_sig` endpoint adds the signature data to the corresponding transaction. It tries to finalize the transaction if it is possible, and returns the same output as the `finalize_tx` endpoint.

### Owner RPC API

### `create_tx`

The `create_tx` endpoint will receive a transaction as a parameter and it will:

- Add the participant data (`excess`, `signature` and `nonce` ).
- Adjust the transaction offset.
- Add the output(s) to the transaction.
- Add the payment proof to the specified transaction
  
It will automatically try to perform a synchronous transaction with the sender, the transaction slatepack is returned if the sender is not reachable. It is recommendable to internally call the `init_send_tx` owner endpoint to unified the experience.

### Commands

### `sign` command

Users will be able of signing the transaction using the `sign` command. This command receives either the `id` or the `UUID` of the transaction, adds the participant data and returns the partially signed slatepack of the transaction.

## Drawbacks
[drawbacks]: #drawbacks

None.

## Unresolved questions
[unresolved-questions]: #unresolved-questions

- Could it be true the default value of the `manual_confirmation` configuration property?
- Will this could work for more than 2 participants?
- Can we replace the `finalize` command by the `sign` command?

## Future possibilities
[future-possibilities]: #future-possibilities

This also adds the ability for the receiver to manually include a payjoin input in a specific transaction but it should be implemented by the wallets developers.

Manually confirmations could be useful if we implement at some point a `memo` field for the `payment proof`. Sender and receiver could agree on accepting a document/file attached to the transaction.

## References
[references]: #references


[1] [Signature Data](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0012-compact-slates.md#signature-data)
