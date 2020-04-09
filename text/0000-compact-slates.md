
- Title: compact-slates
- Authors: [Michael Cordner](mailto:yeastplume@protonmail.com)
- Start date: April 3, 2020 
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [mimblewimble/grin-wallet#366](https://github.com/mimblewimble/grin-wallet/pull/366)

---

# Summary
[summary]: #summary

This RFC describes the changes between version 3 and version 4 of the Slate transaction exchange format, which had the goal of reducing the contents of the Slate to be as minimal as possible.

# Motivation
[motivation]: #motivation

Previously, the definition of Slate versions up to V3 had been put together with no regard for its size or/and redundant/irrelevant content. In order to facilitate future exchange method possibilities, it's desirable to ensure the Slate is as compact as possible, particularly on the 'first leg' of a transaction exchange which only actually requires minimal information from the transaction initiator. 

This RFC aims to define the contents of a streamlined "compact" slate by:

* Removing all redundant or unnecessary Slate fields
* Shortening the names of many Slate fields
* Reducing the size of the Slate to be as minimal as possible, particularly on the first leg

Although this RFC doesn't address any particular transaction exchange methods that might be facilitated by this streamlining, one could envisage possibilities such as:

* An exchange placing the entire initial slate in a QR code
* Encoding the initial slate as an easily-cut-and-paste chunk

# Community-level explanation
[community-level-explanation]: #community-level-explanation

There are two basic transaction workflows in a two-party Grin transaction:

* In the Basic workflow, a sender adds their inputs and change outputs to a transaction, and sends them, along with the fee, the amount and their signature data to the recipient. The recipient adds a output for the amount to the transaction, adds their signature data and returns to the sender, who completes the transaction and posts.

* In the Invoice workflow, the invoice creator adds a new output to a transaction, and sends the amount and signature data to the payer. The payer adds their inputs and change outputs to the transaction along with their signature data and fee information, then returns to the invoicer, who completes the transaction and posts.

Although previous versions of the Slate included every party's complete inputs and outputs at every stage of the transaction, it is not technically necessary for the initiator to provide their inputs and outputs to the other party. It suffices for the initiator to store the input/output listing in their local transaction context, and only provide the amount, fee, excess and signature data to the other party. Recognising this, it is possible to ensure that the 'first-leg' of a transaction stage is extremely compact.

For instance, version 3 of the Slate on transaction initiation may have looked something like the following:

```
{
  "version_info": {
    "version": 3,
    "orig_version": 3,
    "block_header_version": 2
  },
  "num_participants": 2,
  "id": "0436430c-2b02-624c-2032-570501212b00",
  "tx": {
    "offset": "d202964900000000d302964900000000d402964900000000d502964900000000",
    "body": {
      "inputs": [
        {
          "features": "Coinbase",
          "commit": "087df32304c5d4ae8b2af0bc31e700019d722910ef87dd4eec3197b80b207e3045"
        },
        {
          "features": "Coinbase",
          "commit": "08e1da9e6dc4d6e808a718b2f110a991dd775d65ce5ae408a4e1f002a4961aa9e7"
        }
      ],
      "outputs": [
        {
          "features": "Plain",
          "commit": "0812276cc788e6870612296d926cba9f0e7b9810670710b5a6e6f1ba006d395774",
          "proof": "dcff6175390c602bfa92c2ffd1a9b2d84dcc9ea941f6f317bdd0f875244ef23e696fd17c71df79760ce5ce1a96aab1d15dd057358dc835e972febeb86d50ccec0dad7cfe0246d742eb753cf7b88c045d15bc7123f8cf7155647ccf663fca92a83c9a65d0ed756ea7ebffd2cac90c380a102ed9caaa355d175ed0bf58d3ac2f5e909d6c447dfc6b605e04925c2b17c33ebd1908c965a5541ea5d2ed45a0958e6402f89d7a56df1992e036d836e74017e73ccad5cb3a82b8e139e309792a31b15f3ffd72ed033253428c156c2b9799458a25c1da65b719780a22de7fe7f437ae2fccd22cf7ea357ab5aa66a5ef7d71fb0dc64aa0b5761f68278062bb39bb296c787e4cabc5e2a2933a416ce1c9a9696160386449c437e9120f7bb26e5b0e74d1f2e7d5bcd7aafb2a92b87d1548f1f911fb06af7bd6cc13cee29f7c9cb79021aed18186272af0e9d189ec107c81a8a3aeb4782b0d950e4881aa51b776bb6844b25bce97035b48a9bdb2aea3608687bcdd479d4fa998b5a839ff88558e4a29dff0ed13b55900abb5d439b70793d902ae9ad34587b18c919f6b875c91d14deeb1c373f5e76570d59a6549758f655f1128a54f162dfe8868e1587028e26ad91e528c5ae7ee9335fa58fb59022b5de29d80f0764a9917390d46db899acc6a5b416e25ecc9dccb7153646addcc81cadb5f0078febc7e05d7735aba494f39ef05697bbcc9b47b2ccc79595d75fc13c80678b5e237edce58d731f34c05b1ddcaa649acf2d865bbbc3ceda10508bcdd29d0496744644bf1c3516f6687dfeef5649c7dff90627d642739a59d91a8d1d0c4dc55d74a949e1074427664b467992c9e0f7d3af9d6ea79513e8946ddc0d356bac49878e64e6a95b0a30214214faf2ce317fa622ff3266b32a816e10a18e6d789a5da1f23e67b4f970a68a7bcd9e18825ee274b0483896a40"
        }
      ],
      "kernels": [
        {
          "features": "Plain",
          "fee": "7000000",
          "lock_height": "0",
          "excess": "000000000000000000000000000000000000000000000000000000000000000000",
          "excess_sig": "00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"
        }
      ]
    }
  },
  "amount": "60000000000",
  "fee": "7000000",
  "height": "5",
  "lock_height": "0",
  "ttl_cutoff_height": null,
  "payment_proof": null,
  "participant_data": [
    {
      "id": "0",
      "public_blind_excess": "033ac2158fa0077f087de60c19d8e431753baa5b63b6e1477f05a2a6e7190d4592",
      "public_nonce": "031b84c5567b126440995d3ed5aaba0565d71e1834604819ff9c17f5e9d5dd078f",
      "part_sig": null,
      "message": null,
      "message_sig": null
    }
  ]
}
```

In version 4 of the Slate format, the initial Slate now looks like the following:

```
{
  "ver": "4.3",
  "sta": 0,
  "id": "mavTAjNm4NVztDwh4gdSrQ",
  "amt": "1000000000",
  "fee": "8000000",
  "sigs": [
    {
      "excess": "02ef37e5552a112f829e292bb031b484aaba53836cd6415aacbdb8d3d2b0c4bb9a",
      "nonce": "03c78da42179b80bd123f5a39f5f04e2a918679da09cad5558ebbe20953a82883e"
    }
  ]
}
```

The 'return' slate from the recipient to the originator is expected to be larger, as it must include the recipient's outputs/inputs and proofs (as the party who posts the transaction needs all of this information). However, compacting the first-leg journey allows for future possibilities that may have been more difficult in previous iterations of the slate format. For instance, an initiator taking Grin payments might be expected to be better equipped to receive Grin transactions. They could present the payer with an invoice transaction for the amount compacted to the size of a QR code, which the payer could scan, accept, and then post back to the invoicer's listening wallet infrastructure.

Compacting the slate also acts as a minor privacy-enhancer by hiding the initiator's outputs from the other party.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Slate Definition

Entries prefixed with `//` denote fields that may be omitted, as well as their default assumed values

```
{
  "ver": "4.3",
  "sta": 0,
//"num_parts: 2,
  "id": "mavTAjNm4NVztDwh4gdSrQ",
//"tx": null,
  "amt": "1000000000",
  "fee": "8000000",
//"lock_hgt": 0,
//"ttl": null,
  "sigs": [
    {
      "excess": "02ef37e5552a112f829e292bb031b484aaba53836cd6415aacbdb8d3d2b0c4bb9a",
//    "part": null,
      "nonce": "03c78da42179b80bd123f5a39f5f04e2a918679da09cad5558ebbe20953a82883e"
    }
  ]
//"proof": null,
}
```

If included, the proof structure is:

```
  "proof": {
    "saddr": "7e008eb593ba17d116e282d6267a3c6aad87b910933ad34dfa4d7d2c92b6ba31",
//  "rsig": null,
    "raddr": "3a425bd5da5f0f78593251ede7fad0ecf7a95679d84b2cb405255d97ce068234"
  }
```

A description of all fields and their meanings is as follows:

* `ver` - The slate version and supported block header version, separated by a `.`
* `sta` - 2 character String representing the current stage of the the transaction. This field should be completed by each participant before returning the Slate to another party. Valid values are:
   * `S1` - Standard (payer to payee) transaction flow, new transaction that incorporates the sender's inputs and change output(s), if any. Sender's signature nonce, excess (with kernel offset) is included and ready for sending to recipient.
   * `S2` - Standard transaction flow, recipient has created their outputs(s) and supplied their excess, nonce and partial signature, ready to return to recipient for completion.
   * `S3` - Standard transaction flow, Slate complete, contains all inputs, outputs and final signatures, and is ready for posting
   * `I1` - Invoice (payee to payer) transaction flow, new transaction that incorporates the Invoicer's output(s). Invoicer's signature nonce and excess (with kernel offset) is included and ready for sending to Invoicee.
   * `I2` - Invoice workflow, Invoicee has added their inputs and change output(s).  Invoicee's signature, nonce and excess is included and ready for sending back to the invoicer.
   * `I3` - Invoice workflow, Slate complete, contains all inputs, outputs and final signatures, and is ready for posting

* `num_parts` - The option number of participants in the transaction, assumed to be 2 if omitted
* `id` - The slate's UUID, encoded in Base-57 short form
* `tx` - The [Transaction](https://github.com/mimblewimble/grin/blob/34ff103bb02bc093fe73d36641eb193f7ef2404f/core/src/core/transaction.rs#L871); may be omitted during the first part of a transaction exchange
* `amt` - The transaction amount as a string parseable as a u64
* `fee` - The transaction fee as a string parseable as a u64
* `lock_hgt` - Lock height of the transaction (for future use), assumed 0 if omitted
* `ttl` - Time to Live, or block height beyond which wallets should refuse to further process the transaction
* `sigs` - An array of signature data for each participant. Each entry contains
   * `excess` - Hex-String encoded short form public key on the secp256k1 curve representing the public blind excess for the participants inputs/outputs. The first party to add signature data must also generate and add a random kernel offset to this value.
   * `part` - Hex-String encoded Aggregated (Schnorr) secp2561k signature represeting the participant's partial sig. May be omitted if the participant does not yet have enough data to create it
   * `nonce` - The public key of the nonce chosen by the participant for their partial signature
* `proof` - An optional payment proof requet that must be filled out by the recipient if requested (only valid for basic transaction flow). This contains:
   * `saddr` - Hex-String encoded short-form public key on the ed25519 curve, representing the sender's wallet address (see the [payment proofs rfc](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0006-payment-proofs.md) for details.
   * `raddr` - Hex-String encoded short-form public key on the ed25519 curve represnting the recipient's wallet address
   * `rsig` - Hex-String encoded EdDSA ed25519 signature representing the recipient's payment-proof signature. Can be omitted if this has not yet been filled out

### Changes from existing V3 Slate

* The `version_info` struct is removed, and is replaced with `ver`, which has the format <version>.<block header version>
* `id` becomes a short-form base-57 encoding of the UUID
* `amount` is renamed to `amt`
* `height` is removed
* `lock_height` is renamend to `lock_hgt`
* `num_paricipants` is renamed to `num_parts`
* `ttl_cutoff_height` is renamed to `ttl`
*  The `participant_data` struct is renamed to `sigs`
* `public_blind_excess` in a `sigs` entry is renamed to `excess`
* `public_nonce` in a `sigs` entry is renamed to `nonce`
* `part_sig` in a `sigs` entry is renamed to `part`
*  The `payment_proof` struct is renamed to `proof`
*  The `sender_address` field in the `payment_proof (proof)` struct is renamed to `saddr`
*  The `receiver_address` field in the `payment_proof (proof)` struct is renamed to `raddr`
*  The `receiver_signature` field in the `payment_proof (proof)` struct is renamed to `rsig`
* `tx` field becomes an Option
* `tx` field is omitted from the slate if it is None (null)
* `tx` field and enclosed inputs/outputs do not need to be included in the first leg of a transaction exchange. (All inputs/outputs naturally need to be present at time of posting).
* `num_participants (num_parts)` becomes an Option
* `num_participants (num_parts)` may be omitted from the slate if it is None (null), if `num_participants` is omitted, it's value is assumed to be 2
* `lock_height (lock_hgt)` becomes an Option
* `lock_height (lock_hgt)` may be omitted from the slate if it is None (null), if `lock_height` is omitted, it's value is assumed to be 0
* `ttl_cutoff_height (ttl)` may be omitted from the slate if it is None (null),
* `payment_proof (proof)` may be omitted from the slate if it is None (null),
* `message` is removed from `participant_data (sigs)` entries
* `message_sig` is removed from `participant_data (sigs)` entries
* `id` is removed from `participant_data (sigs)` entries. Parties can identify themselves via private keys stored in the transaction context
* `part_sig (part)` may be omitted from a `participant_data (sigs)` entry if it has not yet been filled out
* `receiver_signature (rsig)` may be omitted from `payment_proof (proof)` if it has not yet been filled out

### Pretty-Printing

The examples above are pretty-printed for reference, but all Slates should remove all whitespace before sending to other parties. The whitespace and formatting can be re-added anywhere the Slate needs to be presented for debugging.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Is block header version needed?

# Future possibilities
[future-possibilities]: #future-possibilities

This RFC is envasaged as a necessary first step for all slate-exchange possibilities that would benefit from compactness, e.g:

* [Slate Serialization](https://github.com/j01tz/grin-rfcs/blob/slate-serialization/text/0000-slate-serialization.md)
* QR Code encoding of slates
* Armored slates

# References
[references]: #references

* [Shorter UUIDs](https://github.com/seigert/shorter-uuid-rs)

Include any references such as links to other documents or reference implementations.
