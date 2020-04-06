
- Title: compact-slates
- Authors: [Michael Cordner](mailto:yeastplume@protonmail.com)
- Start date: April 3, 2020 
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [mimblewimble/grin-wallet#366](https://github.com/mimblewimble/grin-wallet/pull/366)

---

# Summary
[summary]: #summary

# Motivation
[motivation]: #motivation

# Community-level explanation
[community-level-explanation]: #community-level-explanation

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

### Slate Definition

Entries prefixed with `//` denote fields that may be omitted, as well as their default assumed values

```
{
  "ver": "4.3",
//"num_parts: 2,
  "id": "mavTAjNm4NVztDwh4gdSrQ",
//"tx": null,
  "amt": "1000000000",
  "fee": "8000000",
  "hgt": "437088",
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
* `num_parts` - The option number of participants in the transaction, assumed to be 2 if omitted
* `id` - The slate's UUID, encoded in Base-57 short form
* `tx` - The [Transaction](https://github.com/mimblewimble/grin/blob/34ff103bb02bc093fe73d36641eb193f7ef2404f/core/src/core/transaction.rs#L871); may be omitted during the first part of a transaction exchange
* `amt` - The transaction amount as a string parseable as a u64
* `fee` - The transaction fee as a string parseable as a u64
* `hgt` - The block height at which the slate was created
* `lock_hgt` - Lock height of the transaction (for future use), assumed 0 if omitted
* `ttl` - Time to Live, or block height beyond which wallets should refuse to further process the transaction
* `sigs` - An array of signature data for each participant. Each entry contains
   * `excess` - The public blind excess for the participants inputs/outputs, (which incorporates the kernel offset created by the first participant to add their signature data
   * `part` - The participant's partial sig, which may be omitted if the participant does not yet have enough data to create it
   * `nonce` - The public key of the nonce chosen by the participant for their partial signature
* `proof` - An optional payment proof requet that must be filled out by the recipient if requested (only valid for basic transaction flow). This contains:
   * `saddr` - The sender's wallet address (see the [payment proofs rfc](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0006-payment-proofs.md) for details
   * `raddr` - The recipient's wallet address
   * `rsig` - The recipient's signature, omitted if this has not yet been filled out

### Changes from existing V3 Slate

* The `version_info` struct is removed, and is replaced with `ver`, which has the format <version>.<block header version>
* `id` becomes a short-form base-57 encoding of the UUID
* `amount` is renamed to `amt`
* `height` is renamed to `hgt`
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

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives


# Prior art
[prior-art]: #prior-art

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Is block header version needed?
- Is height required (it represents the height at which the slate was created, still need to go throughthe code again to recall exactly how it's used

# Future possibilities
[future-possibilities]: #future-possibilities


# References
[references]: #references

* [Shorter UUIDs](https://github.com/seigert/shorter-uuid-rs)

Include any references such as links to other documents or reference implementations.
