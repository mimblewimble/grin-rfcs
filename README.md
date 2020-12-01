# Grin RFCs

This repository contains all [Grin Project](https://grin.mw) RFCs that have been proposed and accepted by the Grin community for further consideration.

Grin RFCs may cover (but are not limited to) technical enhancements, changes to the governance structure or changes to project processes.

## Getting started

To begin writing your own RFC or to find out more about the process and the general RFC guidelines, refer to [the RFC that established this  process](text/0001-rfc-process.md).

## List of accepted RFCs

|Title|Status|Tl;dr|
|:---|---|:---|
| [0001-rfc-process](text/0001-rfc-process.md) | ACTIVE | Introduce RFC process |
| [0002-grin-governance](text/0002-grin-governance.md) | RETIRED | ~~Articulate community values, define core and sub-teams~~ Replaced by RFC#6. |
| [0003-security-process](text/0003-security-process.md) | ACTIVE | Define community standards for ethical disclosure behaviour |
| [0004-full-wallet-lifecycle](text/0004-full-wallet-lifecycle.md) | ACTIVE | Define API standard for sensitive wallet operations |
| [0005-variable-size-kernels](text/0005-variable-size-kernels.md) | ACTIVE | Introduce kernel variants that can be of different sizes |
| [0006-payment-proofs](text/0006-payment-proofs.md) | ACTIVE | Support generating and validating payment proofs for sender-initiated (i.e. non-invoice) transactions |
| [0007-node-api-v2](text/0007-node-api-v2.md) | ACTIVE | Create a v2 JSON-RPC API for the Node API |
| [0008-wallet-state-management](text/0008-wallet-state-management.md) | ACTIVE | Improve wallet state management |
| [0009-enable-faster-sync](text/0009-enable-faster-sync.md) | ACTIVE | Enable faster txhashset sync by changing output MMR commitment |
| [0010-online-transacting-via-tor](text/0010-online-transacting-via-tor.md) | RETIRED | ~~Define standard for transacting via Tor~~ Replaced by RFC#0015. |
| [0011-security-team](text/0011-security-team.md) | ACTIVE | Establish Grin Security team |
| [0012-compact-slates](text/0012-compact-slates.md)| ACTIVE | Introduce new compact slate format (Slate V4) |
| [0013-nrd-kernels](text/0013-nrd-kernels.md) | ACTIVE | Introduce relative timelocks through "No Recent Duplicate" transaction kernels |
| [0014-general-fund-guidelines](text/0014-general-fund-guidelines.md) | ACTIVE | Define general fund spending guidelines |
| [0015-slatepack](text/0015-slatepack.md) | ACTIVE | Universal transaction standard for Grin. Replaces RFC#0010. |
| [0016-simplify-governance](text/0016-simplify-governance.md) | ACTIVE | Simplify Grin governance model. Replaces RFC#0002. |
| [0017-fix-fees](text/0017-fix-fees.md) | ACTIVE | Change minimum relay fees to be weight proportional, make output creation cost 0.01 Grin. |
| [0018-fix-daa](text/0018-fix-daa.md) | ACTIVE | Change DAA (Difficulty Adjustment Algorithm) from damped simple moving average (dsma) to a weighted-target exponential moving average (wtema). Restrict the future-time-limit (FTL) window to 5 minutes. |
| [0019-deprecate-http-tx](text/0019-deprecate-http-tx.md) | ACTIVE | Deprecating HTTP(S) as a transaction method in grin-wallet. |

## License

Apache License 2.0

### Contributions

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you, shall be licensed as above, without any additional terms or conditions.
