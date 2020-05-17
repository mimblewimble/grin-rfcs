- Title: SlatePack
- Authors: [joltz](mailto:joltz@protonmail.com)
- Start date: May 07 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [Edit if merged with link to tracking github issue]
---

# Summary
[summary]: #summary

SlatePack is a universal transaction standard for Grin. It is designed to provide a single coherent transaction building framework to improve both the user and developer experiences in the Grin ecosystem. All wallets and services are expected to fully support the SlatePack standard by the last scheduled hard fork in January 2021 to remain compatible.

This document specifies the required components of the SlatePack standard and introduces them in the context of existing methods for transaction building in Grin. It assumes that SlatePack is the default supported transaction standard for Grin and is intended to operate under all conditions and edge cases. SlatePack is intended to be compatible with the objects and serialization methods defined in the [Slate V4/Compact Slates](https://github.com/mimblewimble/grin-rfcs/pull/49) RFC. This RFC is meant to replace the [Slate Serialization](https://github.com/mimblewimble/grin-rfcs/pull/48), [Armored Slates](https://github.com/mimblewimble/grin-rfcs/pull/53) and [Encrypted Slates](https://github.com/mimblewimble/grin-rfcs/pull/50) RFCs.

# Motivation
[motivation]: #motivation

Without a comprehensive transaction building flow, users and services are left to make their own complicated decisions about firewalls, file handling and compatibility, risking their security, privacy and sanity.

The objective of this RFC is to converge on a simple, universal, adoptable, secure and privacy preserving workflow standard for Grin transactions: SlatePack.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

SlatePack changes the existing transaction building process in Grin in a few ways:

1. _Users, developers and services are no longer required to choose between many possible transaction methods to use and support: SlatePack is a universal Grin transaction standard_
  - The transport method decision now occurs automatically for the user by following the SlatePack standard
  - There is only one synchronous method and one asynchronous method supported by default to keep things simple for developers and support workers

2. _Tor is the only synchronous transaction transport method that is currently supported in the SlatePack standard_
  - This happens "under the hood" by the wallet and the user only has to keep track of a `SlatePack Address` for their counterparty
  - If Tor is not successful, the transaction process automatically falls back to using an encrypted copy and pastable `SlatePack Message` string to complete the transaction asynchronously

3. _The asynchronous method by default is now a copy and pastable `SlatePack Message` string instead of a file_
  - `SlatePack Message` is an ascii-armor string that supports encryption of its payload with a `SlatePack Address`
    - An encrypted `SlatePack Message` is not meaningfully larger than a plain text `SlatePack Message` with regard to transportability as proposed here

4. _The difference between synchronous and asynchronous transaction methods is abstracted away from the end user with the SlatePack standard_
  - `grin-wallet send -d SlatePackAddress 1.337` will first try to send the Grin synchronously via Tor to the `SlatePack Address`
  - If that fails it will fall back to outputting an armored encrypted `SlatePack Message` string for manual copy and paste transport
  - Example `SlatePack Address`: `slatepack1p4fuklglxqsgg602hu4c4jl4aunu5tynyf4lkg96ezh3jefzpy6swshp5x`

5. _Asynchronous transactions are now encrypted by default by knowing the `SlatePack Address` of your counterparty(s)_
  - If a counterparty is unwilling or unable to provide a `SlatePack Address`, a plain text `SlatePack Message` can still be exchanged

6. _Sending a mobile Grin transaction should be as easy as scanning a simple QR encoded from a bech32 `SlatePack Address`_
  - Or as easy as pasting the `SlatePack Address` of your counterparty into your wallet for any other device
  - Or if Tor is not accessible, or the receiving party is not online, as easy as copying and pasting a couple of `SlatePack Message` strings with a counterparty in an alternative communication channel (email, forum, social media, instant messenger, generic web text box, carrier pigeon etc.)

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The SlatePack standard defines the three primary components: `SlatePack Address`, `SlatePack Message` and `SlatePack Workflow`.

The `SlatePack Address` is a shareable bech32 encoded ed25519 public key that can be used both to _route synchronous transactions_ and to _encrypt asynchronous transactions_.

The `SlatePack Message` is an easily copy and pastable ascii-armor string that contains an encrypted slate payload by default and is used in asynchronous transactions.

The `SlatePack Workflow` specifies how both of these components interact in a universally adoptable transaction standard for Grin.

## `SlatePack Address`

A `SlatePack Address` is a bech32 encoded ed25519 public key and when shared with other parties is used to represent the ability to receive Grin transactions.

grin-wallet already handles ed25519 keys for the v3 onion addresses in Tor transactions. These keys can be extended to be a general `SlatePack Address` to allow a universal key format for both transport and encryption that is error-checked, QR friendly and easily human identifiable.

  - Existing ed25519 public keys from the wallet are bech32 encoded with `slatepack` as the `human-readable part` to build a `SlatePack Address`

  - A `SlatePack Address` can be decoded to its ed25519 public key which can then be mapped to an x25519 public key to be used for encryption

 _By default all wallets should generate a new `SlatePack Address` for each transaction for improved user privacy and security._ Wallets can optionally support the ability for a static, reusable receiving `SlatePack Address` with a warning about the privacy risks of reusing these addresses.

ed25519 keys are bech32 encoded as `SlatePack Addresses` rather than x25519 keys because the mapping from ed25519 to x25519 is more straight forward (x25519 public keys do not carry a `v` coordinate so they can map to two possible ed25519 public keys- this is solvable but using the ed25519 as the first order key avoids a potentially complex solution).

#### Key Generation

_The derivation strategy for ed25519 keys used for a `SlatePack Address` in the context of grin-wallet remains open_

#### Example `SlatePack Address`

```
slatepack1p4fuklglxqsgg602hu4c4jl4aunu5tynyf4lkg96ezh3jefzpy6swshp5x
```

## `SlatePack Message`

A `SlatePackMessage` requires multiple layers of data and encoding.

### Serialization

Grin slates are serialized as first order JSON objects. Binary serialization is done on those JSON objects. Before SlatePack, users could use both binary and JSON serialization for asynchronous transactions.

With the SlatePack standard, all asynchronous transactions serialize the slate JSON objects as binary. JSON serialization for synchronous transactions (Tor) is still used as before. The SlatePack standard serialization choices are only relevant for asynchronous transaction methods.

The details for the binary serialization of the most recent slates at the time of this writing can be found in the [Slate V4 (Compact Slates)](https://github.com/mimblewimble/grin-rfcs/pull/49) RFC. Future variations in slate binary serialization should be referenced in an RFC and may require the update of this document.


### Metadata

Metadata is included with SlatePack messages to indicate how to handle the encryption if any for the slata data in addition to tracking versions. It can be expanded in future versions with new fields that are safe to include in plain text. These fields are neither encrypted nor are they cryptographically authenticated by the sender. `mode` indicates which `Metadata` fields are expected to follow.

- `"slatepack": [Major, Minor]`
  - Where `[Major, Minor]` are positive fixnum ints representing the SlatePack version used to build the `SlatePack Message`

- `"mode": int`
  - Where `int` is a positive fixnum int indicating the type of SlatePack message
  - 0 == plain text
  - 1 == encrypted
  - Extendable to future new modes

- `"sender": SlatePackAddress`
  - Where `SlatePackAddress` is a bech32 encoded ed25519 public key generated by the senders wallet
  - For SlatePacks where the user does not wish to provide any `SlatePackAddress` a `0` value is used
  - This value is used in the SlatePack workflow to attempt to complete the transaction via Tor and to otherwise encrypt a slate for asynchronous transport

### Encryption

SlatePack encryption adheres to the cryptography decisions made by [age](https://age-encryption.org). It supports a conversion from the ed25519 signing key type that grin-wallet already uses for Tor to a x25519 encryption key type that age uses for encryption. This allows us to avoid having to make new cryptography decisions to support encrypted slates with keys already used in grin-wallet.

While SlatePack follows age's cryptography decisions exactly, it diverges on _formatting_ choices made by age. This is done because SlatePack asynchronous transactions benefit from a compact format for transport in addition to allowing us to use the JSON building techniques already supported and expected by wallets.

It should also be noted that a `SlatePack Address` could be used to do generic age encryption by decoding the bech32 to the ed25519 public key and mapping that to its corresponding x25519 public key used in age. An `age Address` could also be used as a `SlatePack Address` with some extra effort: bech32 decode to the x25519 public key and then [follow Signal's lead](https://signal.org/docs/specifications/xeddsa/) to attempt to solve the problem of an x25519 key mapping to two ed25519 keys to give a single ed25519 public key to be used to build a `SlatePack Address` by bech32 encoding with `slatepack` as the HRP.

#### Header

A `header` field is included in all encrypted `SlatePack Messages` and contains at minimum the necessary fields for the encrypted `payload` of a SlatePack: `recipients_list` and `mac`.

It is considered part of the `Encryption` layer rather than the `Metadata` layer to help provide a clear distinction of components required for cryptography and truly optional metadata components.

- `recipients_list`

  - For each recipient:

    - `epk` is ephemeral public key `encode(X25519(ephemeral_secret, basepoint))`
      - `encode(data)` is canonical base64 from RFC 4648 without padding.
      - `ephemeral_secret` is `random(32)` and MUST be new for every new `message key`

    - `emk` is encrypted message key `encrypt[HKDF[salt, label](X25519(ephemeral secret, public key))](message key)`
      - `message key` is 128-bit master message key generated by each recipient
      - `encrypt[key](plaintext)` is ChaCha20-Poly1305 from RFC 7539 with a zero nonce.
      - `HKDF[salt, label](key)` is 32 bytes of HKDF from RFC 5869 with SHA-256.
      - `salt` is `X25519(ephemeral secret, basepoint) || public key`
      - `label` is "slatepack_v0.1"

- `mac`
  - A Message Authentication Code is generated on the `header` data which is all of the header values concatenated
    - `encode(HMAC[HKDF["", "header"](message key)](header))`
      - `encode(data)` is canonical base64 from RFC 4648 without padding.
      - `HMAC[key](message)` is HMAC from RFC 2104 with SHA-256.
      - `HKDF[salt, label](key)` is 32 bytes of HKDF from RFC 5869 with SHA-256.

#### Payload (age Enryption)

A binary serialized slate is encrypted according to the age encryption specification. The steps taken here follow age as closely as possible to avoid losing any security properties. An fairly well-review [age library in rust](https://github.com/str4d/rage/tree/master/age) is available to use for implementation.

Any deviations in SlatePack encryption from the exact cryptography steps and decisions made in age are unintentional and should be corrected unless they are explicitly stated as a deviation from the cryptography decisions made by age.

The expected deviations from the age specification are in the formatting and structure of the contents of the file output data (as opposed to the encryption technique itself or the actual binary encrypted payload).

- `payload`
  - `nonce || STREAM[HKDF[nonce, "payload"](message key)](plaintext)`
    - `nonce` is `random(16)`
    - `message key` is 128-bit master message key for each recipient
    - `STREAM` is `ChaCha20-Poly1305` in 64KiB chunks and a nonce structure of 11 bytes of big endian counter, and 1 byte of last block flag (0x00 / 0x01)
    - `HKDF[salt, label](key)` is 32 bytes of HKDF from RFC 5869 with SHA-256

### Armor

The payload that will be armored is a binary serialized `SlatePack` JSON object. Armor is `Framing` wrapped around a `SimpleBase58Check` encoded `Payload`.

#### Framing

Armor uses specific `Headers`, `Footers` and `Periods` as `Framing` to contain its `Payload`.

(Note that the regex for `header` and `footer` below need to reviewed and verified before before FCP)

- `Header`
  - Supported Headers:
    - `BEGINSLATEPACK`
    - `BEGINSLATEPACK X/Y`
      - `Y` is the total number of message parts for a given multipart message and `X` is the number of the current partial message
      - Only used if there is more than one message part
  - Regex: `^[>\n\r\t ]*BEGINSLATEPACK[>\n\r\t ]*([1-9]\/[1-9])*$`


- `Footer`
  - Supported Footers
    - `ENDSLATEPACK`
    - `ENDSLATEPACK X/Y`
      - `Y` is the total number of message parts for a given multipart message and `X` is the number of the current partial message
      - Only used if there is more than one message part
  - Regex: `^[>\n\r\t ]*ENDSLATEPACK[>\n\r\t ]*([1-9]\/[1-9])*$`

- `Periods`

  - All data of an armored slate up to the first `.` is the framing header
  - All data after the first `.` and before the second `.` is the `SimpleBase58Check` encoded payload which contains the slate data
  - All data after the second `.` and before the third `.` is the framing footer
  - Any data after the third `.` is ignored


#### `SimpleBase58Check` Encoding

`SlatePack Message` armor payloads are encoded similar to legacy bitcoin addresses, with the primary differences being that the `SimpleBase58Check` used here does not include version bytes and includes the error checking code at the beginning of the payload instead of at the end.

1. `SHA256(SHA256(SLATEPACK_MESSAGE_BINARY))`

2. First four bytes from previous step are `ERROR_CHECK_CODE`

3. Concatenate `ERROR_CHECK_CODE + SLATEPACK_MESSAGE_BINARY`

4. Base58 encode the output from the previous step to complete the armor `Payload`

It should be noted that the `ERROR_CHECK_CODE` does not have a robust error checking ability because a double sha256 hash is not a proper error check code and the encoding scheme itself was meant to be used for bitcoin addresses which are much smaller than slate payloads.

A more robust error correction option was not chosen here because the consequences of the failure to detect an error are not as severe as they would be for a bitcoin address as further validation would need to occur for Grin. The purpose is to catch some characters being accidentally added or lost during armor transport rather than preventing a spend to an address we don't know the key to spend from.

#### Formatting

- `WORD_LENGTH`: `15`
  - Number of `SimpleBase58Check` encoded characters per word; chosen for human readability across device screen sizes

- `LINE_BREAK`: `200` words
  - Number of words of `WORD_LENGTH` to include before inserting a newline; chosen for user friendliness in terminals and messaging applications

- `MESSAGE_BREAK`: `X` lines
  - Number of lines an armored message formatted with `WORD_LENGTH` and `LINE_BREAK` can contain before it must be split into multiple message parts
  - This parameter still needs to be selected, ideally to cover the most possible likely slate sizes

- `MAX_MESSAGES`: `9`
  - The maximum number of messages that can be used to represent a slate payload

All chosen parameters except for `MAX_MESSAGES` should be changeable without breaking armored slates as whitespace and newlines are removed during decoding and in the case of message breaks, the error check code is over the entire slate payload. `MAX_MESSAGES` cannot exceed 9 because they would be rejected by clients as invalid since the regex will not match. Slates larger than this should use synchronous transport for a better user and developer experience.

### Example SlatePack JSON Object

#### Plain Text

In this plain text example, neither the sender nor the receiver wish to share a `SlatePack Address`

```
{
  "slatepack": [0, 1],
  "mode": 0,
  "sender": "0",
  "payload": <binary serialized slate>
}
```

#### Encrypted

```
{
  "slatepack": [0, 1],
  "mode": 1,
  "sender": "slatepack1p4fuklglxqsgg602hu4c4jl4aunu5tynyf4lkg96ezh3jefzpy6swshp5x",
  "header": {
    "recipients_list": {
        "0": {
            "epk": "SVrzdFfkPxf0LPHOUGB1gNb9E5Vr8EUDa9kxk04iQ0o",
            "emk": "0OrTkKHpE7klNLd0k+9Uam5hkQkzMxaqKcIPRIO1sNE"
        }
    },
    "mac": "gxhoSa5BciRDt8lOpYNcx4EYtKpS0CJ06F3ZwN82VaM"
  },
  "payload": <age encrypted binary seriaized slate binary>,
}
```

### Example `SlatePack Message`

```
BEGINSLATEPACK. 4H1qx1wHe668tFW yC2gfL8PPd8kSgv
pcXQhyRkHbyKHZg GN75o7uWoT3dkib R2tj1fFGN2FoRLY
GWmtgsneoXf7N4D uVWuyZSamPhfF1u AHRaYWvhF7jQvKx
wNJAc7qmVm9JVcm NJLEw4k5BU7jY6S eb. ENDSLATEPACK.
```

## `SlatePack Workflow`

Adoption of the SlatePack standard allows for a unified workflow that can still function without knowledge of a `SlatePack Address` from a counterparty.

### With a `SlatePack Address`

1. `grin-wallet send -d slatepack1bech32address 1.337`

2. Sender wallet derives an onion v3 address from `slatepack1bech32address` and attempts to complete the transaction synchronously via Tor

3. (Fallback) If the synchronous transaction fails, a `SlatePack Message` string is encrypted to the `SlatePack Address` and output for manual asynchronous transport by the user

### Without a `SlatePack Address`

1. `grin-wallet send 1.337`

2. A `SlatePack Message` string is output for manual asynchronous transport by the user

### With QR Codes

A QR-based `SlatePack Workflow` will always begin with a standard QR size because they are encoded directly from a bech32 `SlatePack Address`.

This encoding simultaneously provides a derivable onion address to attempt a synchronous transaction (`bech32 -> ed25519 -> onionv3`) _and_ a derivable encryption key (`bech32 -> ed25519 -> x25519`) to return an encrypted SlatePack string to complete the transaction asynchronously as a fallback.

As a consequence, a `SlatePack` address must be revealed by the party producing a QR code in the `SlatePack Workflow`.

1. Receiver shares `SlatePack Address` via QR

2. Sender scans QR code and the transaction is completed synchronously via Tor by deriving the recipients onion v3 address from their `SlatePack Address`

3. (Fallback) If the synchronous transaction fails, a `SlatePack Message` string is output for manual asynchronous transport by the user

### With Three or More Parties

Some possible future Slatepack transactions may require more than two parties to successfully build. These cases should not require any breaking changes to the core Slatepack standard workflow.

The exact flow order (round-robin etc) will be defined by the accompanying RFCs that define the possible future multiple party slates themselves. In some cases, new slate versions may require (non-breaking) updates to this RFC. From there, the same standard Slatepack standard workflow of attempting to exchange the data via Tor first with an ascii armor fallback is still valid.

For example, a future Slatepack version might add or extend a field to include an array containing a `SlatePack Address` for each party in the order desired to finish building the transaction. The wallet of each subsequent party will attempt to establish a connection with the next via Tor. In the event of a Tor failure it would be the responsibility of the most recent party to manually transport it to the next.

In cases with many parties, the fallback method of the Slatepack standard could quickly become cumbersome if, for example, every third participant fails to achieve a Tor connection.

## Implementation Timeline

1. Initial SlatePack implementation introduced with the _July 2020 hard fork_
  - May or may not support encryption by default yet

2. The proposed SlatePack standard is fully implemented and adopted as a universal transaction standard in _last hard fork (Jan 2021)_
  - SlatePack is the default transaction standard in all wallets and services

# Drawbacks
[drawbacks]: #drawbacks

- This puts a lot of eggs in one basket (if SlatePack fails there will likely be confusion returning to old methods)

- This may be a bit rushed to have where we want it before HF schedule

- Deprecating HTTP(S) is already a major change- by requiring the adoption of this completely new standard in addition we risk putting a lot of effort on the shoulders of existing services in the Grin ecosystem

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- By adding new options without simplifying the workflow for users we risk confusion and friction

- We could just add an option for copy and pastable slates instead of introducing an entirely new universal transaction flow standard

# Prior art
[prior-art]: #prior-art

## Tari

Tari uses the [Tari DHT Network](https://www.tari.com/lessons/02_how_tari_works.html) to support asynchronous mimblewimble transactions. This approach is comprehensive and comprises of the entire peer to peer messaging network, including both nodes and wallets. This is distinct from Slatepack which is strictly an approach to transaction building between wallet software, not general protocol messaging.

Similar to Slatepack, Tari users derive a public key from their master seed (which is represented to users as emojis instead of bech32) and is used to look up peers in peer databases (as opposed to directly routing to a traditional Tor hidden service as in Slatepack). By default Tari, like Slatepack, uses Tor for communication.

While Slatepack and Tari both have addresses that decode to public keys used to find and communicate with counterparty wallets via Tor, they both handle the Tor failure case differently. Tari seems to rely on its custom DHT network to gracefully handle this at the cost of the complexity of a custom DHT layer. Slatepack falls back to an unopinionated, encrypted ascii armor string for the user to transport "outside of the Grin network" to complete the transaction.

The advantage for Slatepack is significantly reduced complexity by using Tor directly with an unopinionated fallback mode. The disadvantage for Slatepack is that transactions don't "magically" just work if Tor communication is failing- they still require some effort from the user to transport the ascii armor themselves.

_Note that these details were taken from early documentation and not code- transactions in Tari may behave differently in practice._

## Beam

Beam uses the [SBBS](https://github.com/BeamMW/beam/wiki/Secure-bulletin-board-system-(SBBS) gossip protocol to support asynchronous mimblewimble transactions. SBBS adds a nontrivial amount of complexity and attack surface to the core Beam software. In exchange, Beam receives a somewhat user-friendly mechanism for users to build transactions asynchronously. The asynchronous fallback method for Slatepack transactions is a simple ascii armor string that does not contain an opinion about a particular protocol with which to exchange the data.

The advantage with the Slatepack method is that much less code is required to support these transactions which can improve the overall stability and security of the codebase running the Grin network. The disadvantage of this for Slatepack is that asynchronous transactions don't "magically" work- they still need to be between users via an outside channel (instant message, text box, email etc). Slatepack makes the tradeoff of slightly more work for the end user in exchange for a simpler and potentially more secure network for Grin.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How to handle key derivation harmoniously?

- What are unmentioned security considerations for using the same base key to both map to an onion address and map to an encryption key used in transactions?
  - Related, what are unmentioned security considerations to `SlatePack Address` reuse?

- What should the `human-readable part` of the bech32 encoded `SlatePack Address` be?
  - `slatepack1p4fuklglxqsgg602hu4c4jl4aunu5tynyf4lkg96ezh3jefzpy6swshp5x`
  - `grin1p4fuklglxqsgg602hu4c4jl4aunu5tynyf4lkg96ezh3jefzpy6swshp5x`
  - `spk1p4fuklglxqsgg602hu4c4jl4aunu5tynyf4lkg96ezh3jefzpy6swshp5x`
  - `sp1p4fuklglxqsgg602hu4c4jl4aunu5tynyf4lkg96ezh3jefzpy6swshp5x`
  - `g1p4fuklglxqsgg602hu4c4jl4aunu5tynyf4lkg96ezh3jefzpy6swshp5x`

- Should we still use double-sha256 in `SimpleBase58Check` or take the opportunity to use a BCH or CRC code which may be more appropriate for error detection on slatepack messages?
  - Is additional engineering desired here if there will always be further validation of the slate payload before a spend can occur?

- If addresses are not reused by default and since wallets need to be able to conduct multiple transactions in parallel, they need the ability to listen on all "active" addresses at the same time

# Future possibilities
[future-possibilities]: #future-possibilities

- Extended to support future modes (payment channel, payjoin, multiple counterparties etc)

- An entirely different standard could be adopted in the future if non-interactive transactions become the default, eliminating the need for SlatePack
  - It might be possible for a new standard to remain compatible with existing `SlatePack Addresses` to allow a more generic `Grin Address`

# References
[references]: #references

https://en.bitcoin.it/wiki/BIP_0173

https://docs.google.com/document/d/11yHom20CrsuX8KQJXBBw04s80Unjv8zCg_A7sPAX_9Y/preview#

https://github.com/str4d/rage/blob/master/age

https://blog.mozilla.org/warner/2011/11/29/ed25519-keys/

https://libsodium.gitbook.io/doc/advanced/ed25519-curve25519

https://blog.filippo.io/using-ed25519-keys-for-encryption/

https://github.com/mimblewimble/grin-rfcs/pull/53

https://saltpack.org

https://github.com/mimblewimble/grin-rfcs/pull/50

https://gitweb.torproject.org/torspec.git/tree/rend-spec-v3.txt

https://signal.org/docs/specifications/xeddsa/

https://www.tari.com/lessons/02_how_tari_works.html

https://github.com/BeamMW/beam/wiki/Secure-bulletin-board-system-(SBBS)
