- Title: armored-slates
- Authors: [joltz](mailto:joltz@protonmail.com)
- Start date: April 24 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

Armored slates allow for an easily copy and pastable string to represent Grin slate data. They can prevent mistakes made while copying and pasting the data by including and verifying an error check code and can be exchanged directly without a file or the need to support Tor or HTTPS. The formatting parameters have been chosen to favor human friendliness and resistance to markdown misinterpretation when copying and pasting.

# Motivation
[motivation]: #motivation

Without armored slates, methods to exchange Grin slates are limited to interactive methods that are complicated to properly configure (HTTPS/Tor) and an inconvenient asynchronous method that requires handling of a file. Armored slates allow a more lightweight and robust method of slate transport: with encoded copy and pastable strings.

Armored slates can support a universally adoptable transaction method that all users can access privately and securely without complex configurations. This greatly reduces the friction previously experienced when trying to support multiple incompatible transaction methods.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

Grin slates are serialized as JSON or as binary by the wallet for transport outside of the wallet. We can add a further step before transport to wrap the slates into ASCII armor to allow them to be more easily exchanged.

Armoring Grin transaction slates means to encode a traditional transaction slate as a copy/pastable string for transport that would otherwise be sent via Tor or https or written to a file. This allows the exchange of slate strings in a similar manner to exchanging a bitcoin address string today. These armored slates can then be decoded by a wallet to be handled like any other slate.

Armored slates have a simple construction. A human readable framing is at the beginning and end of each armored slate (e.g. `BEGINSLATEPACK.` and `. ENDSLATEPACK.`). The framing also contains periods `.` to denote when the payload begins and ends as well as when the message is over.

The payload itself is the serialized slate encoded with a `SimpleBase58Check` encoding similar to bitcoin addresses. This encoding method includes an error check code to ensure that the slate contents have not been accidentally corrupted in transit. Finally the payload is split up into words to make armored slates friendlier for human readability.

Example armored slate:
```
BEGINSLATEPACK. 4H1qx1wHe668tFW yC2gfL8PPd8kSgv
pcXQhyRkHbyKHZg GN75o7uWoT3dkib R2tj1fFGN2FoRLY
GWmtgsneoXf7N4D uVWuyZSamPhfF1u AHRaYWvhF7jQvKx
wNJAc7qmVm9JVcm NJLEw4k5BU7jY6S eb. ENDSLATEPACK.
```

Wallets and services can build a user experience around producing and accepting these armored slates to allow the Grin ecosystem to converge around an acceptable standard transaction method that is not susceptible to the challenges already mentioned with other transaction methods.

Users can pass armored slates to each other in the same mediums used to currently exchange bitcoin addresses.

One way to think about the transaction flow of Grin armored slates is through the lens of a bitcoin transaction flow:
  - Step 1
      - Bitcoin: Alice sends a message to Bob:

      `What is your bitcoin address? I'd like to send you 1.337 btc.`

      - Grin: Alice sends a message to Bob:

      `BEGINSLATEPACK. 4H1qx1wHe668tFW yC2gfL8PPd8kSgv pcXQhyRkHbyKHZg GN75o7uWoT3dkib R2tj1fFGN2FoRLY GWmtgsneoXf7N4D uVWuyZSamPhfF1u AHRaYWvhF7jQvKx wNJAc7qmVm9JVcm NJLEw4k5BU7jY6S eb. ENDSLATEPACK.`

  - Step 2
      - Bitcoin: Bob sends a message to Alice:

      `1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa`

      - Grin: Bob sends a message to Alice:

      `BEGINSLATEPACK. gFiKaEQYKRJu2C1 tqoYgqt6NfMLieE wXWSe186Mi7AUQ6 KvMWUk7wDBDRqZU T3n7eKcHzj7W1CD 4B6zbwZoynadgwH tM6fGJQtFu9yKS3 8EPSHZj2nrSJqQz UTNaPC2NGwWb5is xpbK4odKMLngVJm QNc6m1SM7KvCNRK XkBRFhLPZTEx2t2 2WLjqNsZFvNeQLe jofEkxm3K24csz4 W2uuNA6RSPineET cah4cUWLVqUqUeA 2Qi9Z15o1bbQ9EL 8JMr4nEhav2kPcX 8MH9h1GASX74xNv SPGCVAYrGt1oRHm q84FdJiSAd6PB3x phE65C79oirW93q fqi44FXbwnZzCxJ the7U4mzQNjzNxx ehXy4BVjgPzMyNf h5rA2yWmrHtzqax JW2pjU1T2AF6FLR MmdETC2mYGUvFLz jt3WM8p1X95wv8Z 9MFtgKbujvBV9Nt n8n2W6EAv7misuS uUyJHCaeyaQGtZX wUaqP1thmc9atbQ 6wKRApAErVERvm1 7r2aipvQcvRipFY 4egsqwe516SnNV7 46t2ohaWq3XWEaq 2esvZCx1LKU8eUW NG4EU1WjEXpUfBU MwhhcNpfLbYqLN3 L7pn58eXv5GzhUE mm6B6ui5j14ciMr bhmUXHubhosL3vV BeTJHUi5p9ieE3t wvtu5gzVgXLWmc6 j5DKdWFxMUTLWkt 8ZJQ9wd4sSAUYZa cjnfSoNGhma7SKb JqKv56sCQPZMQiR Z58gF5fF8zNJ8i9 pXB5UPWYEvnEw3Y Fe3WC8esaVwgfYj qqSDgT5MA1ECJdw pSqj4yapJ4YHqrP zY384kBLypdnk8A FTkorpqycqSK3hT GQ6VfHpD7Dxwj9b 4VGU9rrz8TTBppC UzDF5pdW5G2nMZn CTy246iW1bebWMU 31XHL3egWWi7YEj hqsdbet3dtK4qDj bebLHjJBPosQmXq iyWcRSUaWsaDgi8 iQduC8SippvYkcE YudbxJn7zk6ZSYF krxKdi2tSZwq5Xc PuLSmZ6rL2n9vUn 5gt1yCEnJaXCsH5 LGXcAzD2WSsQHbU Rhaj8urGM2FCXWd qGwB8pu9yEhmgND uV9Kt3VgCrGfaAr Rk2BaSd1nbi3gzE rwB2nV2cazbaC5x 3gAtKY4sswA1WsR Z3p52XbK8WjPyxb Ym68EtwQkjAZMm4 bDdB4erjF3XwFn1 T5hgLaFoFymXhbz zejjAdHrRemeHph SW9y4JrZhmsr3fV cUiH32krUVNCqN3 gThkBZNcwqKHmdS tTXBmMi3UzGFHxp 4jM41cVme8R2cU1 iL. ENDSLATEPACK.`

  - Step 3
      - Bitcoin: Alice broadcasts transaction to network
      - Grin: Alice broadcasts transaction to network

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Armored slates are built with two primary pieces. `Framing` wraps a `SimpleBase58Check` encoded slate payload, formatted with adjustable parameters chosen for human usability.

The starting slate data for armoring is a [binary serialized slate](https://github.com/yeastplume/grin-rfcs/blob/compact-slates/text/0000-compact-slates.md#slate-definition---binary). Binary serialized slates allow for more compact armored slates which can translate to an improved user experience.

## Framing

Armor framing allows humans and machines to recognize the beginning and ending of armored slate messages.

### Headers

- `BEGINSLATEPACK`

- `BEGINSLATEPACK X/Y`
  - `Y` is the total number of message parts for a given multipart message and `X` is the number of the current partial message
  - Only used if there is more than one message part

#### Header Regex

`^[>\n\r\t ]*BEGINSLATEPACK[>\n\r\t ]*([1-9]\/[1-9])*$`

### Footers

- `ENDSLATEPACK`

- `ENDSLATEPACK X/Y`
  - `Y` is the total number of message parts for a given multipart message and `X` is the number of the current partial message
  - Only used if there is more than one message part

#### Footer Regex

`^[>\n\r\t ]*ENDSLATEPACK[>\n\r\t ]*([1-9]\/[1-9])*$`

### Periods

Periods (`.`) in the framing mark the data for parsing.

- All data of an armored slate up to the first `.` is the framing header
- All data after the first `.` and before the second `.` is the `SimpleBase58Check` encoded payload which contains the slate data
- All data after the second `.` and before the third `.` is the framing footer
- Any data after the third period is ignored

### Validation

Framing is validated to help ensure that a valid payload is obtained.

1. Collect framing header data by parsing up to the first `.` and assert that it matches the header regex
2. Collect the framing footer data by parsing data between the second and third `.` and assert that it matches the footer regex
  - For multipart messages the message part `X/Y` must match for the header and footer

## `SimpleBase58Check` Encoding

Slate payloads are encoded similar to bitcoin addresses, with the primary difference being the `SimpleBase58Check` used for Grin does not include version bytes as version data can be encoded directly into the slates themselves prior to serialization. Armored slates use binary serialized slates as the starting payload bytes.

  1. The serialized slate bytes are double SHA256 hashed:

  `SHA256(SHA256(SERIALIZED_SLATE_BINARY))`

  2. The first four bytes of the result of the previous step is the:

  `ERROR_CHECK_CODE`

  3. The bytes from step two are concatenated with the bytes from step one:

  `ERROR_CHECK_CODE + SHA256(SHA256(SERIALIZED_SLATE_BINARY))`

  4. The bytes from step three are treated as big-endian and encoded as a base58 string using the same alphabet and mathematical steps as the [`Base58Check`](https://en.bitcoin.it/wiki/Base58Check_encoding) encoding used in bitcoin:

  `BASE58STR1NG`

The resulting `BASE58STR1NG` from the `SimpleBase58Check` encoding is formatted according to the parameter values in the next section. Once the resulting `BASE58STR1NG` is formatted it can be framed as a completed armored slate.

## Formatting Parameters

This section contains details and reasoning for the chosen armor formatting parameters.

- `WORD_LENGTH`: `15`
  - Number of `SimpleBase58Check` encoded characters per word; chosen for human readability across device screen sizes

- `LINE_BREAK`: `200` words
  - Number of words of `WORD_LENGTH` to include before inserting a newline; chosen for user friendliness in terminals and messaging applications

- `MESSAGE_BREAK`: `X` lines
  - Number of lines an armored message formatted with `WORD_LENGTH` and `LINE_BREAK` can contain before it must be split into multiple message parts
  - This parameter still needs to be selected, ideally to cover the most possible likely slate sizes

- `MAX_MESSAGES`: `9`
  - The maximum number of messages that can be used to represent a slate payload

All chosen parameters except for `MAX_MESSAGES` should be changeable without breaking armored slates as whitespace and newlines are removed during decoding and in the case of message breaks, the error check code is over the entire slate payload. `MAX_MESSAGES` cannot exceed `9` because they would be rejected by clients as invalid since the regex will not match. Slates larger than this should use Tor transport for a better user and developer experience.

## Encoding an Armored Slate

The steps below walk through encoding a serialized slate as an armored slate.

```
// Start with bytes of a binary serialized slate as serialized_slate_bytes
// Generate the four byte error code by taking the first four bytes of a
// double sha256 hash of the binary serialized slate bytes
let error_check_bytes = sha256(sha256(serialized_slate_bytes))[0..4];

// Build the payload by concatenating the error check code bytes with
// the serialized slate bytes
let payload_buf = Vec::new();
payload_buf.append(error_check_bytes);
payload_buf.append(serialized_slate_bytes);

// Encode the payload as base58 to complete the last step of the
// `SimpleBase58Check` encoding method
let simple_base58_check = bs58::encode(payload_buf).into_string();

// Format the resulting base58 encoded string to be more human friendly
// formatter() takes arguments for base58_input, word_length, line_break and
// returns a formatted simplebase58check string
// Note that in an actual implementation the formatter() would handle splitting
// messages into multiple parts as necessary
let formatted_payload = formatter(simple_base58_check, 15, 200);

// Build the armor by combining framing and the formatted SimpleBase58Check
// encoded slate
let armor = format!("{}{}{}", "BEGINSLATEPACK. ", formatted_payload, ". ENDSLATEPACK.");

// The armored slate string can now be returned to the user for transport
Ok(armor)
```

## Decoding an Armored Slate

The below steps walk through decoding an armored slate back into the original serialized slate format.

```
// Convert the armored slate from a string to a vector of bytes
let armor_bytes: Vec<u8> = armored_slate.as_bytes().to_vec();

// Collect the bytes up to the first period, this is the framing header
let header_bytes = &armor_bytes
    .iter()
    .take_while(|byte| **byte != b'.')
    .cloned()
    .collect::<Vec<u8>>();

// Verify the header matches the expected regex
assert!(HEADER_REGEX.is_match(str::from_utf8(&header_bytes).unwrap()))

// Collect the payload bytes after the first period until the next period
let payload_bytes = &armor_bytes[header_bytes.len() + 1 as usize..]
    .iter()
    .take_while(|byte| **byte != b'.')
    .cloned()
    .collect::<Vec<u8>>();

// Collect the framing footer bytes until the final period
let footer_bytes = &armor_bytes[header_bytes.len() + payload_bytes.len() + 2 as usize..]
    .iter()
    .take_while(|byte| **byte != b'.')
    .cloned()
    .collect::<Vec<u8>>();

// Verify the footer matches expected regex
assert!(FOOTER_REGEX.is_match(str::from_utf8(&footer_bytes).unwrap()))

// Format payload by removing whitespace
let format_payload = &payload_bytes
    .iter()
    .filter(|byte| !WHITESPACE_LIST.contains(byte))
    .cloned()
    .collect::<Vec<u8>>();

// Decode the SimpleBase58Check payload by first decoding from base58
let base_decode = bs58::decode(&format_payload).into_vec().unwrap();

// Separate the error check and payload bytes
let error_code = &base_decode[0..4];
let slate_bytes = &base_decode[4..];

// Verify the payload bytes by verifying the error check provided is valid
let error_code_check = sha256(sha256(&slate_bytes))[0..4];
assert_eq!(error_code, error_code_check);

// The armor has been removed and the serialized slate bytes can be operated on
Ok(slate_bytes)
```

## Example

In this example we will armor an actual slate according to the rules of this RFC.

1. Generate a binary serialized slate stored in a file with `grin-wallet send -m binfile`
    - This ouputs a file storing the binary serialized slate
    - The actual bytes are:
      - `[0, 4, 0, 3, 120, 208, 191, 149, 249, 149, 79, 56, 136, 148, 187, 170, 211, 119, 39, 247, 1, 6, 0, 0, 0, 0, 79, 177, 0, 64, 0, 0, 0, 0, 0, 106, 207, 192, 1, 0, 3, 76, 29, 45, 39, 151, 148, 10, 25, 152, 182, 250, 220, 26, 214, 165, 170, 78, 219, 67, 226, 40, 14, 174, 252, 163, 152, 227, 230, 70, 144, 167, 118, 3, 127, 107, 57, 241, 153, 184, 238, 46, 194, 208, 17, 34, 81, 196, 164, 197, 36, 103, 240, 37, 206, 237, 207, 114, 236, 126, 108, 86, 11, 89, 115, 120, 0]`

2. Encode the binary serialized slate bytes with `SimpleBase58Check` encoding
    - Generate the error check code
      - Double sha256 slate bytes
        - `SHA256([0, 4, 0, 3, 120, 208, 191, 149, 249, 149, 79, 56, 136, 148, 187, 170, 211, 119, 39, 247, 1, 6, 0, 0, 0, 0, 79, 177, 0, 64, 0, 0, 0, 0, 0, 106, 207, 192, 1, 0, 3, 76, 29, 45, 39, 151, 148, 10, 25, 152, 182, 250, 220, 26, 214, 165, 170, 78, 219, 67, 226, 40, 14, 174, 252, 163, 152, 227, 230, 70, 144, 167, 118, 3, 127, 107, 57, 241, 153, 184, 238, 46, 194, 208, 17, 34, 81, 196, 164, 197, 36, 103, 240, 37, 206, 237, 207, 114, 236, 126, 108, 86, 11, 89, 115, 120, 0])`
          - `[234, 57, 195, 141, 219, 227, 7, 168, 3, 54, 238, 83, 72, 5, 114, 87, 175, 239, 237, 218, 52, 44, 229, 246, 8, 197, 145, 75, 194, 201, 156, 178]`

        - `SHA256([234, 57, 195, 141, 219, 227, 7, 168, 3, 54, 238, 83, 72, 5, 114, 87, 175, 239, 237, 218, 52, 44, 229, 246, 8, 197, 145, 75, 194, 201, 156, 178])`
          - `[37, 137, 61, 119, 97, 235, 87, 190, 175, 160, 85, 62, 33, 149, 254, 114, 225, 146, 33, 82, 182, 119, 130, 116, 208, 100, 167, 80, 171, 134, 146, 163]`

      - Collect first four bytes of the double sha256
        - `[37, 137, 61, 119]`

    - Concatenate the error check code and binary serialized slate bytes
      - `[37, 137, 61, 119, 0, 4, 0, 3, 120, 208, 191, 149, 249, 149, 79, 56, 136, 148, 187, 170, 211, 119, 39, 247, 1, 6, 0, 0, 0, 0, 79, 177, 0, 64, 0, 0, 0, 0, 0, 106, 207, 192, 1, 0, 3, 76, 29, 45, 39, 151, 148, 10, 25, 152, 182, 250, 220, 26, 214, 165, 170, 78, 219, 67, 226, 40, 14, 174, 252, 163, 152, 227, 230, 70, 144, 167, 118, 3, 127, 107, 57, 241, 153, 184, 238, 46, 194, 208, 17, 34, 81, 196, 164, 197, 36, 103, 240, 37, 206, 237, 207, 114, 236, 126, 108, 86, 11, 89, 115, 120, 0]`

    - Encode the concatenated bytes as base58, the result is a `SimpleBase58Check` string
      - `2bcEgR296VvB73ofxoXt1UMnugeTi6EKgt7oCbJjhp6nqA2ZrqewY92upA4g1eFTsqrZM4Vqk7bw7NtUYFdNL5x5GJFEGbjtWXtEVHrMycWHKYiPHnYn5DBEd8vuBdRpcT4QN8bfGuBWR5Tt2BYMMoyR`

3. Format the `SimpleBase58Check` output string to be more human friendly
  - Split words up by adding a space every 15 characters
    - `2bcEgR296VvB73o fxoXt1UMnugeTi6 EKgt7oCbJjhp6nq A2ZrqewY92upA4g 1eFTsqrZM4Vqk7b w7NtUYFdNL5x5GJ FEGbjtWXtEVHrMy cWHKYiPHnYn5DBE d8vuBdRpcT4QN8b fGuBWR5Tt2BYMMo yR`

4. Add framing
  - Frame the output from the previous step with a header and footer and periods
    - `BEGINSLATEPACK. 2bcEgR296VvB73o fxoXt1UMnugeTi6 EKgt7oCbJjhp6nq A2ZrqewY92upA4g 1eFTsqrZM4Vqk7b w7NtUYFdNL5x5GJ FEGbjtWXtEVHrMy cWHKYiPHnYn5DBE d8vuBdRpcT4QN8b fGuBWR5Tt2BYMMo yR. ENDSLATEPACK.`

5. The steps should be reversible to return the original binary serialzied slate bytes

## Handling Edge Cases

Armored slates need to be able to represent any possible valid slate if they are to be considered as a default transaction method. Some slates that contain many outputs, for example a mining pool payout, could potentially become so large that they exceed the maximum armored slate size. In these cases armored slate payloads may require multiple messages. Slate payloads that would exceed the `MESSAGE_BREAK` parameter are split into multiple armored messages:

`BEGINSLATEPACK 1/2. tqoYgqt6NfMLieE ... W2uuNA6RSPineET. ENDSLATEPACK 1/2.`

`BEGINSLATEPACK 2/2. tqoYXHubhosL3vV ... CTy246iW1bebWMU. ENDSLATEPACK 1/2.`

In cases of multiple armored messages, the `ERROR_CHECK_CODE` is over the entire payload, not each individual partial message. Each message should have the same `ERROR_CHECK_CODE` which can be verified once all messages in a series have been received. This ensures that the correct messages are combined in the correct order to produce a valid slate.

## Implementation

A mostly working implementation of armored slates is available at https://github.com/j01tz/slatepack for experimental use.

# Drawbacks
[drawbacks]: #drawbacks

- Armored slates can potentially become too large with slates containing many outputs
  - Multipart messages is a poor user and developer experience

- Implementation can add further complexity to codebase

- May still be too much friction for an average to use when exchanging Grin

- ASCII Armor? What is this, 2007?!

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- PGP Armor encoding is clunky and breaks often with applications misinterpreting some of the base64 characters as markdown, base62 solves this fairly well

- Base58 only loses 1.2% efficiency against base62, in exchange we gain increased ability for humans to type an armored slate by hand if necessary

- Word length is arbitrary as all whitespace is stripped before decoding but 15 characters does well on mobile devices

- No metadata is encoded at armor layer but has been considered, it is probably best to keep the armor layer as simple as possible and include metadata as needed with serialization

- Without armored slates, all Grin users will be forced to use a method that is blocked at the packet level in some countries (Tor), a method that violates privacy (https) or a method that requires accepting binary files from potential strangers (file)

- Why not just use file-based transactions? Accepting binary files from strangers can be risky as malicious bytes can be hidden in the file
  - Even if the wallet perfectly sanitizes all input it is still undesirable to leave potential malicious bytes resting in a file stored indefinitely

- Why not just use JSON strings directly? JSON strings are larger than a binary serialized armored slate, are often broken with markdown interpretations in text fields and do not notify a user if any of the contents are missing are changed as a result of a copy or paste error

# Prior art
[prior-art]: #prior-art

[PGP ASCII Armor](https://tools.ietf.org/html/rfc4880#section-6.2) and [SaltPack ASCII Armor](https://saltpack.org/armoring) formats are the closest examples to Grin Slate Armor. Armored slates more closely resemble SaltPack as some of the user-friendliness strategies have been borrowed from the SaltPack format. However, SaltPack uses MessagePack serialization and multiple encoded packets and Grin armored slates are serialized directly with a [custom binary serialization format](https://github.com/yeastplume/grin-rfcs/blob/compact-slates/text/0000-compact-slates.md#slate-definition---binary).

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- For framing do we want `SLATEPACK`, `GRINSLATE`, `GRINARMOR`, `000000000`?

- What is optimal value for `MESSAGE_BREAK` formatting parameter?

- Is four bytes an optimal size for the error check code?

- Is double sha256 optimal to generate error check code?

- Should armored slates support multiple serialization methods?

- Are armored slates sufficient for a minimum viable transaction including edge cases?

# Future possibilities
[future-possibilities]: #future-possibilities

- Armored slates as the base for a Minimum Viable Transaction

- Armored slates could be encrypted to increase security against surveillance for slate exchange on platforms that do not support end to end encryption

# References
[references]: #references

https://tools.ietf.org/html/rfc4880#section-6.2

https://saltpack.org/armoring

https://en.bitcoin.it/wiki/Base58Check_encoding

https://github.com/mimblewimble/grin-rfcs/pull/49

https://github.com/mimblewimble/grin-rfcs/pull/48
