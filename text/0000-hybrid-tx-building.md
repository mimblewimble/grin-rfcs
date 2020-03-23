
- Title: `hybrid-tx-building`
- Authors: [@lehnberg](mailto:daniel.lehnberg@protonmail.com)
- Start date: `Mar 19, 2020`
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

This RFC establishes a baseline for transaction building in Grin between users and services, through the introduction of a new 'hybrid' method. Transactions are initiated used two standardized formats, **QR code** and **blob**, and can then be completed over Tor or clearnet using https.

To support this, the transaction slate data at each step in the process is redesigned and minified.

The introduction of this method improves security as it does not support unsafe communication methods, and it improves usability as friction and potential drop-off points are reduced in the transaction building flow.

# Background
[background]: #background

## Transaction building at high level

To create valid transactions in Grin, a round-trip of communication between transacting parties is required ("transaction building"). Transaction data ("transaction slates") is exchanged. The round-trip can be carried out in either direction, sender initiated ("default flow") or recipient initiated ("invoice flow").

### Flow overview

#### Default
1. Sender creates a transaction for a certain amount, locks spending outputs, sends data to Recipient (trip 1).
2. Recipient creates destination outputs, produces a response, returns data to Sender (trip 2).
3. Sender finalizes and broadcasts the transaction.

#### Invoice
1. Recipient creates a payment request (an invoice) for a certain amount, creates destination outputs, sends data to Sender (trip 1).
2. Sender locks spending outputs, produces a response, returns data to Recipient (trip 2).
3. Recipient finalizes and broadcasts the transaction.

### Previous slate versions

The transaction slates up until this point (v0-3) have been serialized as JSON objects, making it easy to read for both humans and machines. The structure of data passed at each step is described in detail in the grin-wallet documentation.<sup>[[1]](#1)</sup> The intention to that point was for slates to contain a complete picture of the transaction state at every stage. It was not a priority to limit the information shared or minimizing the payload transmitted in each communication round.

# Motivation
[motivation]: #motivation

## Transaction building pain points

After more than a year of mainnet transactions between users and services like mining pools and exchanges, observations include the following pain points.

### NAT / Firewall restrictions

Often behind NAT or a firewall, users can only reliably be expected to initiate connections. This complicates receiving payments from services. Users often end up having to tunnel through trusted third party services<sup>[[2]](#2)</sup> that relay their communication, which introduces privacy and security concerns.

→ How can users behind NAT and Firewalls be better supported?

### Transacting is not mobile-friendly

It's challenging to run a http listener service on mobile wallets, making it difficult to receive payments. While file based transacting is possible, files are not always handled well on mobile phone OSes, and it can be difficult to exchange data between makes and models without relying on third party services.

→ How can the experience for users on mobile devices be improved?

### Lack of asynch support

Current http(s) based transaction methods expect synchronous communication when no third party relaying service is used. While it can be expected of businesses to be constantly available and listening for inbound connections, it's not realistic to expect the same from end users. This often leads to failures in the transaction building process, resulting in poor user experience when outputs become locked for transactions that do not finalize.

→ How can the requirement for fully synchronous communication be relaxed?

### Insecure default methods

Since http communication is supported as a transaction building method, many services default to using that as it is straight forward to implement. This un-encrypted communication can easily be intercepted or spoofed and introduces considerable security risks. The argument for deprecating the http method for end users has been made a long time<sup>[[3]](#3)</sup>, but has so far been unconvincing as there's not been a good enough solution presented as an alternative.

→ What alternative can encourage services to replace http as their default?

### Tor solves many issues, but is an awkward default

Support for transaction building over the Tor overlay network was introduced as part of RFC#0010.<sup>[[4]](#4)</sup> In addition to disguising the IP addresses of the transacting parties, Tor solves the problem of accessing users behind NAT or Firewalls, and is end-to-end encrypted by design. It is therefore a good solution for many users, especially those with formidable threat models. It is not a good solution for all however, as it is blocked in some regions and/or may not culturally be accepted to be used by companies and end users. While there are methods to bypass the technological restrictions, they add complexity and are not guaranteed to always work. Futhermore, although Tor access on mobile is possible, it is far from trivial and makes mobile wallet development more complicated. More generally, it creates an external dependency on Tor being operational in order for the transaction building process to function. For these reasons, there's a need for falling back to other methods when Tor is unavailable or is not suitable.<sup>[[5]](#5)</sup>

→ Tor is a good solution that should be used when it is appropriate. What is a good fallback alternative that works in a wider range of conditions?

## Transaction building using a hybrid approach

The pain points above can be addressed by handling the two communication trips in the transaction building process differently. Standard rules and formats are enforced for the first trip where the slate is passed as a QR code or a blob of text. This slate then includes instructions for completing the second trip over https.

### Use case examples

**End users** are assumed to use mobile or desktop wallets, being behind NAT or Firewalls, and to come online intermittently.

**Online services** (such as exchanges, mining pools, or online stores) are assumed to be available and able to listen for inbound connections as required.

#### User purchases item in online store using grin

* On checkout, store generates QR code and presents to the browser window of the user's laptop.
* User scans the QR code with their mobile camera. A URI triggers the Grin mobile wallet, which processes the message and displays a confirmation window with the amount and destination URL. User taps to confirm.
* Store receives the response, finalizes the transaction, and broadcasts it.

#### User deposits grin to exchange

* At the exchange deposit screen, user makes a deposit request for a given amount. The exchange generates a QR code and presents to the browser window of the user's laptop.
* User scans the QR code with their mobile camera. A URI triggers the Grin mobile wallet, which processes the message and displays a confirmation window with the amount and destination URL. User taps to confirm.
* Exchange receives the response, finalizes the transaction, and broadcasts it.


#### User withdraws grin from mining pool

* User logs into the pool, makes a withdrawal request for a certain amount. The mining pool generates a blob that is presented in the browser window of the user.
* User copies the blob and pastes it into their wallet. Wallet processes the message and displays a confirmation window with the amount and the sender URL. User taps to confirm.
* Mining pool receives the response, finalizes the transaction and broadcasts it.

### Benefits

* **A universal format for transactions that is flexible.** The hybrid approach supports Tor and clearnet communication interchangeably in trip 2, following an identical process that is seamless to the user. It can be extended to support other methods as required without altering the flows or formats.

* **Bypasses NAT & Firewall restrictions.** The QR code or blob in trip 1 is delivered through an existing channel, and since this includes instructions for opening an outbound connection to complete trip 2, NAT & Firewall issues are not relevant.

* **Asynchronous.** Communication in the two trips is broken up, and the transacting party that receives the slate as a QR code or blob does not need to respond immediately. The initiator is however still required to listen on the designated trip 2 response address.

* **Mobile friendly.** A mobile wallet user can scan the QR code with their phone, or quickly copy/paste the blob from a chat message, and open an outbound connection to complete the transaction without having to run a listener at any time.

* **An improved user experience.**
   * **Fewer friction points.** The notion of addresses can be abstracted, and the process can become much more streamlined: Users can transact by scanning a code with their phone and confirming with a single tap.
   * **Fewer drop-off points.** This method is less error-prone, as it makes no assumptions of wallet connectivity in trip 1.
   * **Reduced locked output limbo.** When a service is the initiating party, spending outputs of end users stay locked for a shorter duration. 
   * **More descriptive errors.** In those events where transacting still fails, wallet developers can use the additional information provided in the slate of trip 1 to present the user with better descriptive error messages.

* **Encrypted communication by default.** By enforcing https in the response address, encrypted communication becomes the standard. As the hybrid method improves usability over http transacting, it stands a chance to become adopted as a new default method to transact with services.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

## Overview
The main contribution of this RFC is the introduction a novel way to pass data in trip 1 of transaction building. 

Slate information is packaged into two standardized formats that are interchangeable:
* A **QR code image**, which can be read by the camera of a mobile device, making it cross-device portable: The slate can be presented on one device and read from a different.
* A **blob**, which is a string of characters that can be copy/pasted and shared over any medium. It's similar to the file exchange method, without requiring the handling of an actual file.

The slate in trip 1 includes an instruction to the wallet of the other party for how to communicate in trip 2 over an https address in order to complete the round trip, alongside an optional TTL for how long this address will be valid.

### Flows

#### Hybrid method, default flow
1. Sender creates a transaction for a certain amount, locks spending outputs, chooses a reply-to https address, creates a QR code or blob that is sent to Recipient (trip 1).
2. Recipient processes the QR code or blob, creates destination outputs, produces a response, responds to Sender at the designated address (trip 2).
3. Sender finalizes and broadcasts the transaction.

#### Hybrid method, invoice flow
1. Recipient creates a payment request (an invoice) for a certain amount, creates destination outputs, chooses a reply-to https address, creates a QR code or blob that is sent to Sender (trip 1).
2. Sender processes the QR code or blob, locks spending outputs, produces a response, responds to Sender at the designated address (trip 2).
3. Recipient finalizes and broadcasts the transaction.


<!-- Explain the proposal as if it were already introduced into the Grin ecosystem and you were teaching it to another community member. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Grin community members should *think* about the improvement, and how it should impact the way they interact with Grin. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Grin community members and new Grin community members.

For implementation-oriented RFCs (e.g. for wallet), this section should focus on how wallet contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms. -->

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The technical details of this RFC can be broken down into three distinct efforts:

* **Step 1:** Redesigning the transaction slate to ensure that only required data is passed at every stage in the communication process, and add new fields for the hybrid method.
* **Step 2:** Change the serialization of transaction slates to make the slate footprint as small as possible.
* **Step 3:** Define two standards for creating the first trip of communication: a QR code format and a text blob format.

## 1. New slate structure

### Design approach

* Only required information included at each phase
* Treat defaults as implicit

### New fields

#### Reply-to https address

#### TTL

### Description of new flows

| Stage | Standard | Invoice |
| --- | --- | ---|
Init |
Slate<br> trip 1 |
Receive | 
Slate <br>trip 2 |
Finalize |

## 2. Slate serialization

## 3. Standards requirements

### QR code

### Blob


# Drawbacks
[drawbacks]: #drawbacks

<!-- Why should we *not* do this? -->
- Adds complexity?
- Does it solve the problem well?

Tbd

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

* Tor transacting only?
* File transacting only?

<!-- - Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this? -->
Tbd

# Prior art
[prior-art]: #prior-art

<!-- Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For core, node, wallet and infrastructure proposals: Does this feature exist in other projects and what experience have their community had?
- For community, ecosystem and moderation proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture. If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other projects.

Note that while precedent set by other projects is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that Grin sometimes intentionally diverges from common project features. -->

* Grinbox

* https://github.com/mimblewimble/grin-wallet/issues/67

# Unresolved questions
[unresolved-questions]: #unresolved-questions

<!-- * - What parts of the design do you expect to resolve through the RFC process before this gets merged?
* What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
* What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC? -->
 
- Does this make it easier to transact user <-> user? How do two mobile users transact?
- What happens to file exchange? Should we consider supporting it in trip 2?
- What are the implications of adding TTL? Are there negative consequences? 
- Where does the keybase transaction building method fit in all this? Should it be deprecated?

# Future possibilities
[future-possibilities]: #future-possibilities

<!-- * Think about what the natural extension and evolution of your proposal would be and how it would affect the project and ecosystem as a whole in a holistic way. Try to use this section as a tool to more fully consider all possible interactions with the project and language in your proposal. Also consider how it fits into the road-map of the project and of the relevant sub-team.

* This is also a good place to "dump ideas", if they are out of scope for the RFC you are writing but otherwise related.

* If you have tried and cannot think of any future possibilities, you may simply state that you cannot think of anything.

* Note that having something written down in the future-possibilities section is not a reason to accept the current or a future RFC; such notes should be in the section on motivation or rationale in this or subsequent RFCs. The section merely provides additional information. -->

* Open ended transaction payment request
* Payment proofs for invoice flows
* NFC standard
* End to end encrypt blobs / QR codes? Password protect them?

# References
[references]: #references

<a name="1"></a>[1]: https://github.com/mimblewimble/grin-wallet/blob/master/doc/transaction/basic-transaction-wf.png

<a name="2"></a>[2]: Examples of such tunnels include [ngrok](https://ngrok.com), Grin++ Relay (URL needed), and [Hedwig](https://forum.grin.mw/t/niffler-an-out-of-the-box-open-sourced-grin-gui-wallet/4760/6).

<a name="3"></a>[3]: https://github.com/mimblewimble/grin-wallet/issues/66

<a name="4"></a>[4]: https://github.com/mimblewimble/grin-rfcs/blob/master/text/0010-online-transacting-via-tor.md

<a name="5"></a>[5]: For a detailed discussion, see https://github.com/j01tz/network-protocol-obfuscation/blob/master/grin_obfuscation.md