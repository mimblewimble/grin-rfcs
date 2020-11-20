- Title: Deprecate HTTP(S) Transactions
- Authors: [joltz](mailto:joltz@protonmail.com)
- Start date: April 28 2020
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary
[summary]: #summary

Deprecating HTTP(S) as a transaction method helps preserve privacy for Grin users during the slate exchange process while building transactions. By only allowing the http listener in `grin-wallet` to listen on localhost by default, users and services will be encouraged to use more privacy-friendly transaction methods like Tor or directly exchanging armored slates via the [Slatepack](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0015-slatepack.md) standard. This improves privacy for the Grin ecosystem by ensuring that services do not force Grin users into transaction methods that can easily violate their privacy like HTTP(S).

# Motivation
[motivation]: #motivation

Previously, Grin slates could be exchanged over HTTP(S) as a convenient method to take the required steps to exchange slate data between wallets to build a transaction. This method was commonly adopted by exchanges and services who already had infrastructure in place to configure wallets to listen for incoming HTTPS requests.

Unfortunately, while convenient, this method of exchange requires the sender and receiver to reveal their IP addresses to route the network packets. This is bad for user privacy because without taking extra obfuscating steps, an IP address can often be traced to an individual.

While steps can be taken to obfuscate a users true IP (like using a VPN), it is undesirable for a privacy-preserving technology like Grin to rely on this method for conventional use. It does not seem natural for users that are concerned about their privacy enough to use a technology like Grin to at the same time accept that their IP address is revealed when exchanging Grin with many services.

Deprecating HTTP(S) provides a few benefits:
  - User experience is improved by not having to manage firewall settings like port forwarding to accept Grin transactions from most users
  - Wallet developer experience is improved with the need to support fewer transaction methods
  - Users will no longer be encouraged to engage in transaction methods that potentially violate privacy during the transaction building process
  - The ecosystem can converge around a universally accepted transaction flow that is privacy preserving

The overall expected outcome is a privacy improvement for users that may otherwise have to reveal their IP address to send or receive Grin.

# Community-level explanation
[community-level-explanation]: #community-level-explanation

Deprecating HTTP(S) means that the old method of sending and listening for transactions over traditional HTTP(S) is no longer supported to help preserve privacy for Grin users who may not be aware of the risk to privacy using HTTP(S) poses.

This reduces the overall number of transaction transport methods that need to be supported but requires that all services and wallets that previously only accepted/sent HTTP(S) transactions to upgrade at least to the version this RFC is implemented in to support their users.

For the average user this means that to send or receive Grin they can use the [Slatepack](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0015-slatepack.md) method. It is no longer possible to send and receive Grin via the HTTP(S) method without custom configuration.

For wallet developers, it is no longer necessary to be concerned about managing HTTP(S) endpoints. All transactions will be conducted via Tor or asynchronous exchange of armored slate strings with the Slatepack standard. It can no longer be assumed that the counterparty in the transaction supports HTTP(S).

Previously, some services like exchanges _only_ supported HTTP(S) Grin transactions. This change impacts these services significantly.

Slatepack with Tor is the closest replacement to HTTP(S), but for some services the use of Tor is unacceptable, not possible or not allowed. These services will will need to refactor the user flow to allow supporting manual copying and pasting of [Slatepack armored transaction messages](https://github.com/mimblewimble/grin-rfcs/blob/master/text/0015-slatepack.md#slatepackmessage) instead, as the HTTP(S) method is no longer supported for `grin-wallet`.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The technical and implementation details for this RFC are simple: no new features are added and some existing features are restricted:

- `grin-wallet` http listener only accepts connections from localhost

- `grin-wallet listen` no longer accepts `-e` argument

- `grin-wallet send -d` requires a valid Slatepack address

## Deprecation Timeline

- With v4.0.0 a public announcement will be made to notify the Grin ecosystem that HTTP(S) will be fully deprecated in v5.0.0

- With v4.0.0 wallets will warn users that their privacy may be at risk when sending or listening via HTTP(S) and that the method will be deprecated in next major release

- With v5.0.0 wallets will no longer support sending transactions via HTTP(S)

- With v5.0.0 wallets will only accept connections from localhost by default.

# Drawbacks
[drawbacks]: #drawbacks

- Forces changes to existing transaction building workflows that use HTTP(S)

- Remaining transaction options (Slatepack via Tor and armored message) may not be appealing for services already exclusively using HTTP(S)

- Forces "always on" wallet listeners to use Tor or to build out logic to support handling armored slates in the UX flow

- Potentially makes a technology that is already challenging for services to adopt even more challenging to adopt

- Those that do not value privacy as a first-order objective may see this as an unnecessary inconvenience

- Friction could cause some services to avoid Grin completely rather than adding support for a new transaction method

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

By not deprecating HTTP(S) an opportunity is lost to provide an additional layer of privacy to present and future Grin supporters.

Deprecating HTTP only while continuing to support HTTPS would not likely be viable. As currently implemented, removing HTTP support would also mean removing HTTPS support. Allowing HTTPS while blocking HTTP may beyond the scope of `grin-wallet` configuration.

# Prior art
[prior-art]: #prior-art

A [general comparison of transaction methods for Grin](https://github.com/mimblewimble/grin-pm/issues/283) is available and provides many considerations when weighing possible transaction methods, including HTTP(S).

Bitcoin previously used a similar transaction method (send to IP) which was deprecated as well for many valid reasons. While the arguments presented in this RFC are primarily derived from the objective of delivering privacy, there are other good reasons previously discussed in the context of Bitcoin. These discussions can be found partially in the [bitcointalk post](https://bitcointalk.org/index.php?topic=9334.0) accompanying the [PR](https://github.com/bitcoin/bitcoin/pull/253) that removed this feature for Bitcoin.

Note that Grin is not quite in the same position as Bitcoin was when they deprecated this method as an alternative method existed that did not require interaction was available (P2PKH). Grin will still require interaction for the foreseeable future, either directly via Tor or asynchronously via Slatepack armored messages.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Is it acceptable to the ecosystem to remove a commonly used transaction method?

- Will the adoption of this RFC cause an unacceptable loss of support from existing adopting services?

- Will deprecating HTTP(S) grow or reduce adoption in the next 6-12 months? 1-3 years? 5-10 years?

# Future possibilities
[future-possibilities]: #future-possibilities

This RFC along with others can eventually support ecosystem convergence around a single acceptable transaction building workflow supported by all wallets and services.

# References
[references]: #references

https://bitcointalk.org/index.php?topic=9334.0

https://github.com/bitcoin/bitcoin/pull/253

https://github.com/mimblewimble/grin-pm/issues/283

https://github.com/mimblewimble/grin-rfcs/blob/master/text/0015-slatepack.md
