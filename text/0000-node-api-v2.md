
- Title: Node API V2
- Authors: [Quentin Le Sceller](mailto:q.lesceller@gmail.com)
- Start date: September 30th, 2019
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#0000](https://github.com/mimblewimble/grin-rfcs/pull/0000) 
- Tracking issue: [Edit if merged with link to tracking github issue]

---

# Summary

[summary]: #summary

Create a v2 JSON-RPC API for the Node API.

# Motivation

[motivation]: #motivation

The current Node API (referred here as v1 API) is a REST API. This API while simple is:

- Documented manually (current documentation is available [here](https://github.com/mimblewimble/grin/blob/master/doc/api/node_api.md).
- Contains call with heterogenous args such as `?byid=xxx` and `commitment/xxx` which can be confusing and lack some uniformity.
- Uses REST which is bound to HTTP while v2 wallet API uses JSON-RPC.

The goal of this RFC is to provide a new API with cleaner methods and errors, automatically generated documentation directly on docs.rs and finally stronger basis for future improvements.

# Community-level explanation

[community-level-explanation]: #community-level-explanation

This new API will be particularly useful for wallet developer and should ultimately simplify their work on Grin. Moreover, this RFC does not introduce any breaking changes as the v1 REST API will still be around until completely deprecated.

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

## Current Endpoints with the v1 API

While the endpoints are documented in details [here](https://github.com/mimblewimble/grin/blob/master/doc/api/node_api.md), here is an overview of the current REST API Endpoints of the V1 API.

```JSON
[
  "get blocks",
  "get headers",
  "get chain",
  "post chain/compact",
  "get chain/validate",
  "get chain/kernels/xxx?min_height=yyy&max_height=zzz",
  "get chain/outputs/byids?id=xxx,yyy,zzz",
  "get chain/outputs/byheight?start_height=101&end_height=200",
  "get status",
  "get txhashset/roots",
  "get txhashset/lastoutputs?n=10",
  "get txhashset/lastrangeproofs",
  "get txhashset/lastkernels",
  "get txhashset/outputs?start_index=1&max=100",
  "get txhashset/merkleproof?n=1",
  "get pool",
  "post pool/push_tx",
  "post peers/a.b.c.d:p/ban",
  "post peers/a.b.c.d:p/unban",
  "get peers/all",
  "get peers/connected",
  "get peers/a.b.c.d",
  "get version"
]
```

This endpoints can be grouped into 5 categories:

- miscellaneous endpoints (which contain `status` and `version` endpoints)
- `chain` endpoints (which also contain `blocks`, `chain` and headers` endpoints)
- `peer` endpoints
- `pool` endpoints
- `txhashset` endpoints

## Proposed Endpoints

### Miscellaneous endpoints

#### get_status

Returns various information about the node, the network and the current sync status.

```JSON
{
  "jsonrpc": "2.0",
  "method": "get_status",
  "params": [],
  "id": 1
}
{
"id": 1,
  "jsonrpc": "2.0",
  "result": {
    "Ok": {
      "protocol_version": "2",
      "user_agent": "MW/Grin 2.x.x",
      "connections": "8",
      "tip": {
        "height": 371553,
        "last_block_pushed": "00001d1623db988d7ed10c5b6319360a52f20c89b4710474145806ba0e8455ec",
        "prev_block_to_last": "0000029f51bacee81c49a27b4bc9c6c446e03183867c922890f90bb17108d89f",
        "total_difficulty": 1127628411943045
      },
      "sync_status": "header_sync",
      "sync_info": {
        "current_height": 371553,
        "highest_height": 0
      }
    }
  }
}
```

#### get_version

Returns the node version and block header version (used by grin-wallet).

```JSON
{
  "jsonrpc": "2.0",
  "method": "get_version",
  "params": [],
  "id": 1
}
{
"id": 1,
  "jsonrpc": "2.0",
  "result": {
    "Ok": {
      "node_version": "2.1.0-beta.2",
      "block_header_version": 2
    }
  }
}
```

### Chain endpoints

TBD

### Peer endpoints

TBD

### Pool endpoints

TBD

### TxHashSet endpoints

TBD

## Authentication

Similarly to the V1 API, the v2 API will use basic auth with the same secret. This token is usually in `grin/main/.api_secret`.

### Wallet support

The wallet currently uses the Grin Node API v1 so changes will be needed in the grin-wallet repository to make it compatible with the v2 Node API.

### Legacy support

The V1 API will remain active for a time the mode of operation for its REST API will be assumed to work as currently. This setup should allow existing wallets and apps to continue working as-is until a cutoff release for legacy mode is determined.

### API only

Note that this RFC does not propose making user-facing changes to the existing CLI (invoked by `grin client`) to invoke these functions. It's expected that the existing cli functionality will be modified to invoke the new API functions.

# Drawbacks

[drawbacks]: #drawbacks

The implementation of this RFC will temporarily introduces some additional code complexity as v1 and v2 node API will need to coexist until a cutoff release is determined.

# Prior art

[prior-art]: #prior-art

This kind of JSON-RPC API is widely used for cryptocurrencies. For instance:

- [Bitcoin](https://en.bitcoin.it/wiki/API_reference_\(JSON-RPC\))
- [Ethereum](https://github.com/ethereum/wiki/wiki/JSON-RPC)
- [Monero](https://web.getmonero.org/resources/developer-guides/wallet-rpc.html)
- [Tezos](https://tezos.gitlab.io/master/developer/rpc.html)

# Future possibilities

[future-possibilities]: #future-possibilities

This new API will simplify the deployment of new methods and will drastically simplify the work of developers by providing a clear documentation directly on docs.rs.

# References

[references]: #references

- [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)
- [Bitcoin JSON-RPC Doc](https://en.bitcoin.it/wiki/API_reference_\(JSON-RPC\))
- [Ethereum JSON-RPC Doc](https://github.com/ethereum/wiki/wiki/JSON-RPC)
- [Monero JSON-RPC Doc](https://web.getmonero.org/resources/developer-guides/wallet-rpc.html)
- [Tezos JSON-RPC Doc](https://tezos.gitlab.io/master/developer/rpc.html)
