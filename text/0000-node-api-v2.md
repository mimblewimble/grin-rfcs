
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

#### get_header

Gets block header given either a height, a hash or an unspent output commitment.
Only one parameters is needed. If multiple parameters are provided only the first one in the list is used.

```JSON
{
    "jsonrpc": "2.0",
    "method": "get_header",
    "params": [null, "00000100c54dcb7a9cbb03aaf55da511aca2c98b801ffd45046b3991e4f697f9", null],
    "id": 1
}
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": {
    "Ok": {
      "cuckoo_solution": [
        9886309,
        35936712,
        43170402,
        48069549,
        70022151,
        97464262,
        107044653,
        108342481,
        118947913,
        130828808,
        144192311,
        149269998,
        179888206,
        180736988,
        207416734,
        227431174,
        238941623,
        245603454,
        261819503,
        280895459,
        284655965,
        293675096,
        297070583,
        299129598,
        302141405,
        313482158,
        321703003,
        351704938,
        376529742,
        381955038,
        383597880,
        408364901,
        423241240,
        436882285,
        442043438,
        446377997,
        470779425,
        473427731,
        477149621,
        483204863,
        496335498,
        534567776
      ],
      "edge_bits": 29,
      "hash": "00000100c54dcb7a9cbb03aaf55da511aca2c98b801ffd45046b3991e4f697f9",
      "height": 374336,
      "kernel_root": "d294e6017b9905b288dc62f6f725c864665391c41da20a18a371e3492c448b88",
      "nonce": 4715085839955132421,
      "output_root": "12464313f7cd758a7761f65b2837e9b9af62ad4060c97180555bfc7e7e5808fa",
      "prev_root": "e22090fefaece85df1441e62179af097458e2bdcf600f8629b977470db1b6db1",
      "previous": "0000015957d92c9e04c6f3aec8c5b9976f3d25f52ff459c630a01a643af4a88c",
      "range_proof_root": "4fd9a9189e0965aa9cdeb9cf7873ecd9e6586eac1dd9ca3915bc50824a253b02",
      "secondary_scaling": 561,
      "timestamp": "2019-10-03T16:08:11+00:00",
      "total_difficulty": 1133587428693359,
      "total_kernel_offset": "0320b6f8a4a4180ed79ecd67c8059c1d7bd74afe144d225395857386e5822314",
      "version": 2
    }
  }
}
```

#### get_block

Gets block details given either a height, a hash or an unspent output commitment.
Only one parameters is needed. If multiple parameters are provided only the first one in the list is used.

```JSON
{
    "jsonrpc": "2.0",
    "method": "get_block",
    "params": [374274, null, null],
    "id": 1
}
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": {
    "Ok": {
      "header": {
        "cuckoo_solution": [
          1263501,
          14648727,
          42430559,
          58137254,
          68666726,
          72784903,
          101936839,
          104273571,
          123886748,
          131179768,
          155443226,
          162493783,
          164784425,
          167313215,
          169806918,
          183041591,
          184403611,
          210351649,
          215159650,
          239995384,
          240935454,
          257742462,
          280820644,
          300143903,
          303146496,
          311804841,
          341039986,
          354918290,
          363508555,
          377618528,
          396693709,
          397417856,
          399875872,
          413238540,
          413767813,
          432697194,
          436903767,
          447257325,
          453337210,
          459401597,
          496068509,
          511300624
        ],
        "edge_bits": 29,
        "hash": "000001e16cb374e38c979c353a0aaffbf5b939da7688f69ad99efda6c112ea9b",
        "height": 374274,
        "kernel_root": "e17920c0e456a6feebf19e24a46f510a85f21cb60e81012f843c00fe2c4cad6e",
        "nonce": 4354431877761457166,
        "output_root": "1e9daee31b80c6b83573eacfd3048a4af57c614bd36f9acd5fb50fbd236beb16",
        "prev_root": "9827b8ffab942e264b6ac81f2b487e3de65e411145c514092ce783df9344fa8a",
        "previous": "00001266a73ba6a8032ef8b4d4f5508407ffb1c270c105dac06f4669c17af020",
        "range_proof_root": "3491b8c46a3919df637a636ca72824377f89c4967dcfe4857379a4a82b510069",
        "secondary_scaling": 571,
        "timestamp": "2019-10-03T15:15:35+00:00",
        "total_difficulty": 1133438031814173,
        "total_kernel_offset": "63315ca0be65c9f6ddf2d3306876caf9f458a01d1a0bf50cc4d3c9b699161958",
        "version": 2
      },
      "inputs": [],
      "kernels": [
        {
          "excess": "08761e9cb1eea5bfcf771d1218b5ec802798d6eecaf75faae50ba3a1997aaef009",
          "excess_sig": "971317046c533d21dff3e449cc9380c2be10b0274f70e009aa2453f755239e3299883c09a1785b15a141d89d563cdd59395886c7d63aba9c2b6438575555e2c4",
          "features": "Coinbase",
          "fee": 0,
          "lock_height": 0
        }
      ],
      "outputs": [
        {
          "block_height": 374274,
          "commit": "09d33615563ba2d65acc2b295a024337166b9f520122d49730c73e8bfb43017610",
          "merkle_proof": "00000000003e6f5e000000000000000f60fe09a7601a519d9be71135404580ad9de0964c70a7619b1731dca2cd8c1ae1dce9f544df671d63ff0e05b58f070cb48e163ca8f44fb4446c9fe1fc9cfef90e4b81e7119e8cf60acb9515363ecaea1ce20d2a8ea2f6f638f79a33a19d0d7b54cfff3daf8d21c243ba4ccd2c0fbda833edfa2506b1b326059d124e0c2e27cda90268e66f2dcc7576efac9ebbb831894d7776c191671c3294c2ca0af23201498a7f5ce98d5440ca24116b40ac98b1c5e38b28c8b560afc4f4684b81ab34f8cf162201040d4779195ba0e4967d1dd8184b579208e9ebebafa2f5004c51f5902a94bf268fd498f0247e8ba1a46efec8d88fa44d5ecb206fbe728ee56c24af36442eba416ea4d06e1ea267309bc2e6f961c57069e2525d17e78748254729d7fdec56720aa85fe6d89b2756a7eeed0a7aa5d13cfb874e3c65576ec8a15d6df17d7d4856653696b10fb9ec205f5e4d1c7a1f3e2dd2994b12eeed93e84776d8dcd8a5d78aecd4f96ae95c0b090d104adf2aa84f0a1fbd8d319fea5476d1a306b2800716e60b00115a5cca678617361c5a89660b4536c56254bc8dd7035d96f05de62b042d16acaeff57c111fdf243b859984063e3fcfdf40c4c4a52889706857a7c3e90e264f30f40cc87bd20e74689f14284bc5ea0a540950dfcc8d33c503477eb1c60",
          "mmr_index": 4091742,
          "output_type": "Coinbase",
          "proof": "7adae7bcecf735c70eaa21e8fdce1d3c83d7b593f082fc29e16ff2c64ee5aaa15b682e5583257cf351de457dda8f877f4d8c1492af3aaf25cf5f496fce7ca54a0ef78cc61c4252c490386f3c69132960e9edc811add6415a6026d53d604414a5f4dd330a63fcbb005ba908a45b2fb1950a9529f793405832e57c89a36d3920715bc2d43db16a718ecd19aeb23428b5d3eeb89d73c28272a7f2b39b8923e777d8eb2c5ce9872353ba026dc79fdb093a6538868b4d184215afc29a9f90548f9c32aa663f9197fea1cadbb28d40d35ed79947b4b2b722e30e877a15aa2ecf95896faad173af2e2795b36ce342dfdacf13a2f4f273ab9927371f52913367d1d58246a0c35c8f0d2330fcddb9eec34c277b1cfdaf7639eec2095930b2adef17e0eb94f32e071bf1c607d2ef1757d66647477335188e5afc058c07fe0440a67804fbdd5d35d850391ead3e9c8a3136ae1c42a33d5b01fb2c6ec84a465df3f74358cbc28542036ae4ef3e63046fbd2bce6b12f829ed193fb51ea87790e88f1ea686d943c46714b076fb8c6be7c577bca5b2792e63d5f7b8f6018730b6f9ddaf5758a5fa6a3859d68b317ad4383719211e78f2ca832fd34c6a222a8488e40519179209ad1979f3095b7b7ba7f57e81c371989a4ace465149b0fe576d89473bc596c54cee663fbf78196e7eb31e4d56604c5226e9242a68bda95e1b45473c52f63fe865901839e82079a9935e25fe8d44e339484ba0a62d20857c6b3f15ab5c56b59c7523b63f86fa8977e3f4c35dc8b1c446c48a28947f9d9bd9992763404bcba95f94b45d643f07bb7c352bfad30809c741938b103a44218696206ca1e18f0b10b222d8685cc1ed89d5fdb0c7258b66486e35c0fd560a678864fd64c642b2b689a0c46d1be6b402265b7808cd61a95c2b4a4df280e3f0ec090197fb039d32538d05d3f0a082f5",
          "proof_hash": "cfd97db403c274220bb0dbaf3ecc88e483c0b707d8e6f16dfda37cd4f2c3211c",
          "spent": false
        }
      ]
    }
  }
}
```

#### get_tip

Returns details about the state of the current fork tip.

```JSON
{
    "jsonrpc": "2.0",
    "method": "get_tip",
    "params": [],
    "id": 1
}
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": {
    "Ok": {
      "height": 374350,
      "last_block_pushed": "000000543c69a0306b5463b92939643442a44a6d9be5bef72bea9fc1d718d310",
      "prev_block_to_last": "000001237c6bac162f1add2b122fab6a254b9fcc2c4b4c8c632a8c39855521f1",
      "total_difficulty": 1133621604919005
    }
  }
}
```

#### validate_chain

Trigger a validation of the chain state.

```JSON
{
    "jsonrpc": "2.0",
    "method": "validate_chain",
    "params": [],
    "id": 1
}
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": {
    "Ok": null
  }
}
```

#### compact_chain

Trigger a compaction of the chain state to regain storage space.

```JSON
{
    "jsonrpc": "2.0",
    "method": "compact_chain",
    "params": [],
    "id": 1
}
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": {
    "Ok": null
  }
}
```


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
