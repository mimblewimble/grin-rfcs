
- Title: slate-serialization
- Authors: [joltz](mailto:joltz@protonmail.com)
- Start date: March 31 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

This RFC proposes SlatePack, an extendable MessagePack-based slate-serialization specification for Grin inspired by SaltPack to standardize and improve Grin slate transportability and support future extensions for signed and encrypted slates. This slate serialization standard ensures compatibility and interoperability for Grin transaction slates in all wallets without wallet developers having to guess what form an incoming slate will take and users from having to keep track of whether the way their wallet serializes slates is compatible with the format of the wallet their counterparty is using.

* *MessagePack* is an efficient binary serialization format, similar to JSON but smaller. https://msgpack.org
* *SaltPack* is a robust modern crypto binary messaging format that uses MessagePack for encoding and NaCl for cryptography. https://saltpack.org
* *SlatePack* is a Grin slate binary messaging format that uses MessagePack for encoding and SaltPack for cryptography decisions

# Motivation
[motivation]: #motivation

It can be beneficial to serialize the Slate data structure used in grin-wallet into a more robust and transportable format. By using SlatePack serialization rather than plain json, we can reduce the minimum required size of the slate data to pass between wallets to better support QR codes, armored slates and encrypted slates. By standardizing this, we can ensure that wallets and services will support a universal and efficient encoding for slates.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

SlatePack serialization allows all Grin wallets to transact with each other at a low level without worrying about compatibility. All wallets are able to encode and decode slates with the same technique to ensure that their users can exchange slates with users of any other wallet.

Previously, slate exchange was somewhat standardized by serializing into json, but json can be bulky and challenging to exchange the data. Now slates have a more efficient json-friendly SlatePack standard for serialization. MessagePack has many well-supported libraries implemented in many languages and is more compact than json in transit which allows transaction slates to require less data to represent them, allowing things like QR codes and copying and pasting slate strings without needing to manage entire files. The work required to follow the SlatePack standard beyond using MessagePack can usually be done in a few lines of code (see next section).

Wallet developers need to be aware of this layer so that they can be prepared to handle incoming and outgoing slate data. Wallets that use grin-wallet already will have this serialization support built-in so should not need to do anything to support SlatePack. Wallets that are based on their own implementation will need to find the MessagePack library in their language and implement it in addition to taking note of the new `HeaderPacket` and `PayloadPacket` in the section below so that they are prepared to serialize outgoing slates and deserialize incoming slates.

Previously Slate Objects were serialized as json strings. In the case of file-based transactions, these json strings were then encoded as bytes.

V3 Blank Slate Application Object representation in grin-wallet:
```
Slate { version_info: VersionCompatInfo { version: 3, orig_version: 3, block_header_version: 3 }, num_participants: 2, id: 7a316cf0-3262-406d-89df-925fdf81ff7b, tx: Transaction { offset: BlindingFactor(<secret key hidden>), body: TransactionBody { inputs: [], outputs: [], kernels: [] } }, amount: 0, fee: 0, height: 0, lock_height: 0, ttl_cutoff_height: None, participant_data: [], payment_proof: None }
```
Here is the v3 blank slate serialized as a json string (old way):
```
"{\"version_info\":{\"version\":3,\"orig_version\":3,\"block_header_version\":3},\"num_participants\":2,\"id\":\"7a316cf0-3262-406d-89df-925fdf81ff7b\",\"tx\":{\"offset\":\"0000000000000000000000000000000000000000000000000000000000000000\",\"body\":{\"inputs\":[],\"outputs\":[],\"kernels\":[]}},\"amount\":\"0\",\"fee\":\"0\",\"height\":\"0\",\"lock_height\":\"0\",\"ttl_cutoff_height\":null,\"participant_data\":[],\"payment_proof\":null}"
```
Here is a v3 blank slate as a string after being serialized with the new method (HeaderPacket not included as it would be twice-encoded):
```
"[[3,3,3],2,\"7a316cf0-3262-406d-89df-925fdf81ff7b\",[\"0000000000000000000000000000000000000000000000000000000000000000\",[[],[],[]]],\"0\",\"0\",\"0\",\"0\",null,[],null]"
```
Much less data is required to represent the Slate Object.

Below is the byte representation of the old json serialization (which is how slates were previously represented in file-based transactions):
```
[123, 34, 118, 101, 114, 115, 105, 111, 110, 95, 105, 110, 102, 111, 34, 58, 123, 34, 118, 101, 114, 115, 105, 111, 110, 34, 58, 51, 44, 34, 111, 114, 105, 103, 95, 118, 101, 114, 115, 105, 111, 110, 34, 58, 51, 44, 34, 98, 108, 111, 99, 107, 95, 104, 101, 97, 100, 101, 114, 95, 118, 101, 114, 115, 105, 111, 110, 34, 58, 51, 125, 44, 34, 110, 117, 109, 95, 112, 97, 114, 116, 105, 99, 105, 112, 97, 110, 116, 115, 34, 58, 50, 44, 34, 105, 100, 34, 58, 34, 55, 97, 51, 49, 54, 99, 102, 48, 45, 51, 50, 54, 50, 45, 52, 48, 54, 100, 45, 56, 57, 100, 102, 45, 57, 50, 53, 102, 100, 102, 56, 49, 102, 102, 55, 98, 34, 44, 34, 116, 120, 34, 58, 123, 34, 111, 102, 102, 115, 101, 116, 34, 58, 34, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 34, 44, 34, 98, 111, 100, 121, 34, 58, 123, 34, 105, 110, 112, 117, 116, 115, 34, 58, 91, 93, 44, 34, 111, 117, 116, 112, 117, 116, 115, 34, 58, 91, 93, 44, 34, 107, 101, 114, 110, 101, 108, 115, 34, 58, 91, 93, 125, 125, 44, 34, 97, 109, 111, 117, 110, 116, 34, 58, 34, 48, 34, 44, 34, 102, 101, 101, 34, 58, 34, 48, 34, 44, 34, 104, 101, 105, 103, 104, 116, 34, 58, 34, 48, 34, 44, 34, 108, 111, 99, 107, 95, 104, 101, 105, 103, 104, 116, 34, 58, 34, 48, 34, 44, 34, 116, 116, 108, 95, 99, 117, 116, 111, 102, 102, 95, 104, 101, 105, 103, 104, 116, 34, 58, 110, 117, 108, 108, 44, 34, 112, 97, 114, 116, 105, 99, 105, 112, 97, 110, 116, 95, 100, 97, 116, 97, 34, 58, 91, 93, 44, 34, 112, 97, 121, 109, 101, 110, 116, 95, 112, 114, 111, 111, 102, 34, 58, 110, 117, 108, 108, 125]
```
Below is the byte representation of the new slate object serialized with SlatePack from the v3 blank slate object from above (including HeaderPacket and PayloadPacket):
```
[196, 15, 147, 169, 115, 108, 97, 116, 101, 112, 97, 99, 107, 146, 0, 1, 4, 145, 155, 147, 3, 3, 3, 2, 217, 36, 99, 49, 52, 101, 49, 48, 102, 48, 45, 50, 53, 101, 101, 45, 52, 51, 56, 50, 45, 97, 50, 102, 99, 45, 51, 54, 101, 50, 55, 48, 52, 102, 97, 53, 100, 52, 146, 217, 64, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 48, 147, 144, 144, 144, 161, 48, 161, 48, 161, 48, 161, 48, 192, 144, 192]
```
In this case, SlatePack is ~286% more efficient than the previous method when serialization a blank v3 slate.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

To implement the SlatePack standard for slate serialization, wallets will likely use MessagePack libraries to help serialize the Slate object for transport, as there are many existing well-reviewed implementations. The MessagePack specification is available at https://github.com/msgpack/msgpack/blob/master/spec.md.

Slates are not serialized directly into MessagePack. Instead two new objects are created: a `HeaderPacket` which contains additional fields for `format_name` to identify the format for the message and `version` and `mode` for future extensions. A `PayloadPacket` object is also created which currently contains `slate` data. The `HeaderPacket` is double-encoded with MessagePack (once as an array object, again as a bin object) and the `PayloadPacket` is encoded as a MessagePack array object. The `HeaderPacket` and `PayloadPacket` are concatenated to form the serialized slate.

Note that many design choices were inspired by https://saltpack.org. It may make sense to adhere more or less to their specification for Grin. For encrypted slates it is best to follow their specification as much as possible to avoid damaging assumed security properties.

However, the closer Grin slates adhere to the SaltPack specification, the more complex the serialization becomes. After review it may be determined that the added complexity is not worth encrypted slates and signatures and a pure MessagePack or minimized SlatePack serialization technique may be adopted for efficiency reasons only.

## Serialization Packets
[serialization-packets]: #serialization-packets

Slate serialization requires two packet types: `Header` and `Payload`. With two distinct packet types, as opposed to naively encoding the slate application object directly into a MessagePack object, we can potentially gain some beneficial security properties in future versions: improved privacy (against man-in-the-middle), authenticity (tamper-resistant slates) and more.

Below is an example for new Rust applications objects that might be used for serializing a slate:
```
// Version to track support for extended header/payload fields
struct SlatePackVersion(u32, u32); // Major, minor

// Mode to indicate the payload that will follow
enum Mode {
  PlainText = 0, // Plain text is encoded directly
  Encryption = 1, // Not in use
  AttachedSigning = 2, // Not in use
}

// This will be twice-encoded during serialization
struct HeaderPacket {
    format_name: String, // "slatepack"
    version: SlatePackVersion,
    mode: Mode,
}

// This will be extended to support encrypted slates
struct PayloadPacket {
    slate: Slate, // Slate application object
}
```

### Header Packets
[header-packets]: #header-packets

`HeaderPackets` include three fields for now: `format_name`, `version` and `mode`. More can be added in the future to support features like encrypted slates.

`format_name` "slatepack" unless it makes more sense to use "saltpack" and ensure compatibility via versioning (if going this method, further steps should be taken to ensure SaltPack compatibility beyond the currently proposed SlatePack). This section should be updated to reflect the standard chosen for serialized-slates before acceptance.

`version` is to track what features a client is expected to support when receiving a serialized slate. It could replace existing slate version tracking or supplement it. This field is needed at this layer because if encrypted slates are used in the future, a `version` only stored in the slate data itself would not be accessible (and a client wouldn't be able to know how to handle the encrypted data).

`mode` tracks what kind serialized slate we are passing. There is currently only one mode (plain text), but in the future `modes` for encryption, signatures, lightning channels and more can be added.

Finally, Header packets are twice-encoded: first as a messagepack `array` object and then again as a messagepack `bin` object.

### Payload Packets
[payload-packets]: #payload-packets

`PayloadPackets` currently only have one field: `slate`. In the future more fields may be added to support encrypted slates. Payload packets are serialized as messagepack `array` objects.

*Important Note: The SlatePack version specified in this RFC does not currently support multiple payload packets and the payload packet size is limited to 1MB. If Grin wishes to support slate data >1MB we should consider adding support for multiple PayloadPackets by including a `final_flag` field.*

## Serializing
[serializing]: #serializing

This section contains steps for following the proposed SlatePack standard when serializing Slate Objects.

1. Prepare `HeaderPacket`
  - Create a new Header Packet object including `format_name`, `version` and `mode`
  - Serialize the application object as messagepack `array` object
  - Serialize the `array` object again as a messagepack `bin` object
    - In the future the `array` object will be hashed and used in Payload Packets for increased security
2. Prepare `PayloadPacket`
  - Create a new Payload Packet application object containing the Slate application object
  - Serialize Payload Packet application object as a messagepack `array` object
3. Concatenate `HeaderPacket` and `PayloadPacket` to complete the slate serialization
  - (SerializedHeaderPacket || SerializedPayloadPacket)
  - This is now a fully serialized slate

Steps in pseudocode:
```
// Create new SlatePack HeaderPacket
let header_packet = HeaderPacket::new("slatepack", (0, 1), 0);

// Create new PayloadPacket         
let payload_packet = PayloadPacket::new(slate);    

// Serialize HeaderPacket as MessagePack array object
let mpk_array_header = messagepack::Serialize::array(header_packet);

// Re-Serialize HeaderPacket as MessagePack bin object
let mpk_bin_header = messagepack::Serialize::bin(mpk_array_header);

// Serialize PayloadPacket as MessagePack array object
let mpk_payload = messagepack::Serialize::array(payload_packet);  

// Concatenate HeaderPacket and PayloadPacket
let serialized_slate = concat!(mpk_bin_header, mpk_payload);   
```

## Deserializing
[deserialized]: #deserialized

This section contains steps for following the proposed SlatePack standard when deserializing Slate Objects.

1. Deserialize `HeaderPacket`
  - Use messagepack function to read the first `bin` object length
  - Read the Header Packet `bin` object from the buffer
  - Use messagepack function to decode the `bin` object into an `array` object
  - From the `array` object you can further deserialize into a json string that your application is familiar with (or in some cases deserialize directly into your application object)
  - From here you can access the Header Packet in its native application form for further parsing and validation
  - The Header Packet `mode` may inform how to handle the Payload Packet
2. Deserialize `PayloadPacket`
  - Once the `HeaderPacket` has been consumed, you can deserialize the remaining messagepack `array` object which is the `PayloadPacket`
  - The messagepack `array` object can be deserialized into whatever is convenient for your native application objects
  - Currently only slates exist in the payload packet, in the future necessary fields for encrypted slates will be added

Steps in pseudocode:
```
// Obtain the length of the HeaderPacket from SlatePack
let header_len = messagepack::Deserialize::read_bin_len(serialized_slate_reader.as_bytes());

// Read for the length of the HeaderPacket
// The header is double-encoded so make sure to skip the "bin" tag at the front
let header_packet = buf.read_exact(header_len);

// Deserialize the messagepack bin object into a messagepack array object
let header_deserialize = messagepack::Deserialize::from_bin(header_packet).as_array();

// Deserialize the messagepack array object into application object
let header_app_obj: HeaderPacket = return_object(header_deserialize);

// Deserialize the remaining messagepack array object as the PayloadPacket
let payload_app_obj: PayloadPacket = return_object(serialized_slate_reader);

// Access the deserialized slate as an application object
let deserialized_slate: Slate = payload_app_obj.slate;
```

## Future Versions
[future-versions]: #future-versions

In future versions `HeaderPackets` and `PayloadPackets` can be extended to include support for encryption, signing and more. Compatibility is important so clients need awareness of versioning to enable as much backward compatibility as possible.

# Drawbacks
[drawbacks]: #drawbacks

- JSON is already widely used and likely doesn't add dependencies to the codebase
- SlatePack adds some complexity during serialization (especially if we want encrypted slates)
- SlatePack arguably does not produce "minimal" slates
- SlatePack is less performant in serializing/deserializing large files (though that is less relevant for our usecase)
- SlatePack requires new (but well supported) dependencies unless we implement MessagePack from scratch
- Even if we want encrypted and signed slates we may not want NaCl to supply the crypto!
  - SlatePack may not be the most sensible specification to follow if we don't wish to use NaCl to support crypto to encrypted slates

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

SlatePack was chosen for standardizing Grin slate serialization for a few reasons:
  - MessagePack doesn't go far enough
    - We need more than pure MessagePack encoding to allow future support for encrypted and signed slates
    - We need a way to indicate version of the serialization standard to allow for future changes without requiring changes to the Slate object
  - SaltPack goes too far
    - Grin slates do not yet need all of the features that SaltPack offers
    - Does not support plain text messages without signatures
    - Size is important for some slate transport methods and SlatePack can produce smaller slates than SaltPack
  - SlatePack preserves flexibility for future unexplored uses like opening Lightning channels
  - Not having a well-specified standard for Grin slate serialization can lead to incompatibilities between wallets and frustration between users
  - The old serialization was less efficient and more challenging for QR transport

This RFC includes features like twice-encoded header packets to be included with serialization to allow future extensions. However it may be desirable to keep things as simple as possible and _only_ use messagepack serialization directly on the existing slate and deal only with plain messagepack objects. This still gives us the reduced size benefits but will make it more challenging in the future to add support for encrypted slates (because they would not be backward compatible). It would also make armoring more complicated as some additional headers will likely be encoded at that layer if not at here at the serialization layer.

## Alternative serialization methods considered:
[alternative-serialization-methods-considered]: #alternative-serialization-methods-considered

- MessagePack (no modifications): https://msgpack.org
  - Interchangeable with json, which we already use
  - Compacts the size of slate data to be transmitted
  - Used in SaltPack, which may be adopted in the future for encrypted slates/payment proofs
  - Has many libraries available in many languages
  - Fair number of other projects already use
  - Ongoing development, specification will update
- Cap'n Proto: https://capnproto.org/
  - Can't really take advantage of the serialization perf benefits
  - Well used rust implementation https://github.com/capnproto/capnproto-rust
  - Smaller community than MessagePack
- protobuf: https://developers.google.com/protocol-buffers
  - Similar perf but: msgpack = protocol implemented by many libs, protobuf = libs from a single vendor
- CBOR: https://cbor.io/
  - Very similar to msgpack but has opinions that can complicate encoding/parsing, is standardized (ietf's bikeshedded msgpack)
- BSON: http://bsonspec.org/
  - Not enough of an efficiency gain with initial tests
- AGE (not a serialization method but mentioned for relevance): https://github.com/FiloSottile/age
  - Relevant quote from FV: the age armor is more meant for non 8-bit clean environments (like PowerShell, and yeah maybe online forums and stuff) than for messages, because age is a file encryption tool, not an email or message encryption tool.

# Prior art
[prior-art]: #prior-art

Most cryptocurrencies do not require slate interactivity, so there are not known existing serialization standards for transaction slates. Lightning Network however does require interactivity. Their solution is engineered for something slightly different though- we are not trying to solve for all messages between nodes but for slate messages between wallets.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What are potential future performance bottlenecks?
- What are potential future compatibility bottlenecks?
- Do we want to faithfully follow SaltPack's specification to keep compatibility?
- Is there other slate data that makes sense to be moved to `HeaderPackets`?
- Are the benefits worth the complexity?
- Should a `network` field be added to `HeaderPacket` to distinguish mainnet/testnet/forknet slates?

# Future possibilities
[future-possibilities]: #future-possibilities

The proposed SlatePack serialization standard may eventually support armored and encrypted slates as well as more robust payment proofs.

Transport methods and keys can also be added in an updated version of this RFC. They would likely be encoded in the `PayloadPacket` as new fields for `TransportMethod` and `TransportKey`. This would allow a user to receive a slate in any channel (for example QR) had have the remaining legs of the trip automatically executed via pre-agreed transport method. This could potentially remove an extra step of friction and improve UX.

SlatePack may be the first step on the road to a "Minimum Viable Transaction" guaranteed to work with any wallet, user or service using Grin.

# References
[references]: #references

https://msgpack.org

https://github.com/msgpack/msgpack/blob/master/spec.md

https://en.wikipedia.org/wiki/Comparison_of_data-serialization_formats

https://saltpack.org
