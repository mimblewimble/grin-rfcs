
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

#### get_kernel

Returns a `LocatedTxKernel` based on the kernel excess. The `min_height` and `max_height` parameters are both optional. If not supplied, `min_height` will be set to 0 and `max_height` will be set to the head of the chain. The method will start at the block height `max_height` and traverse the kernel MMR backwards, until either the kernel is found or min_height is reached.

```JSON
{
    "jsonrpc": "2.0",
    "method": "get_kernel",
    "params": ["09c868a2fed619580f296e91d2819b6b3ae61ab734bf3d9c3eafa6d9700f00361b", null, null],
    "id": 1
}
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": {
    "Ok": {
      "height": 374557,
      "mmr_index": 2211662,
      "tx_kernel": {
        "excess": "09c868a2fed619580f296e91d2819b6b3ae61ab734bf3d9c3eafa6d9700f00361b",
        "excess_sig": "1720ec1b94aa5d6ba4d567f7446314f9a6d064eea69c5675cc5659f65f290d80b0e9e3a48d818cadba0a4e894bbc6eb6754b56f53813e2ee0b1447969894ca4a",
        "features": "Coinbase"
      }
    }
  }
}
```

#### get_outputs

Retrieves details about specifics outputs. Supports retrieval of multiple outputs in a single request. Support retrieval by both commitment string and block height. Last field are for whether or not the response will include rangeproof and merkle proof.

```JSON
{
    "jsonrpc": "2.0",
    "method": "get_outputs",
    "params": [["09bab2bdba2e6aed690b5eda11accc13c06723ca5965bb460c5f2383655989af3f","08ecd94ae293863286e99d37f4685f07369bc084ba74d5c59c7f15359a75c84c03"],376150, 376154, true, true],
    "id": 1
}
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": {
    "Ok": [
      {
        "block_height": 374568,
        "commit": "09bab2bdba2e6aed690b5eda11accc13c06723ca5965bb460c5f2383655989af3f",
        "merkle_proof": null,
        "mmr_index": 4093403,
        "output_type": "Transaction",
        "proof": "e30aa961d6f89361a9a3c60f73e3551f50a3887212e524b5874ac50c1759bb95bc8e588d82dd51d84c7cbaa9abe79e0b8fe902bcfda17276c24d269fbf636aa2016c65a760a02e18338a33e83dec8e51fbfd953ee5b765d97ce39ba0850790d2104812a1d15d5eaa174de548144d3a7d413906d85e22f89065ef727910ee4c573494520c43e36e83dacee8096666aa4033b5e8322e72930c3f8476bb7be9aef0838a2ad6c28f4f5212708bf3e5954fc3971d66b7835383b96406fa65415b64ecd53a747f41d785c3e3615c18dfdbe39a0920fefcf6bc55fe65b4b215b1ad98c80fdafbef6f21ab60596f2d9a3e7bc45d750e807d5eb883dadde1625d4f20af9f1315b8bea08c97fad922afe2000c84c9eb5f96b2a24da7a637f95c1102ecfc1257e19bc4120082f5ee76448c90abd55108256f8341e0f4009cfc3906a598de465467ee1ee072bfd3384e1a0b9039192d1edc33092d7b09d1164c4fc4c378227a391600a8a5d5ba5fe36a2a4eabe0dbae270aefa5a5f2df810cda79211805206ad93ae08689e2675aad025db3499d43f1effc110dfb2f540ccd6eb972c02f98e8151535c099381c8aeb1ea8aad2cfdf952e6ab9d26e74a5611d943d02315e212eb06ce2cd20b4675e6f245e5302cdb8b31d46bb2e718b50ecfad2d440323826570447c2498376c8bad6e4ee97bde41c47f6a20eea406d758c53fb9e8542f114c1a277a6335ad97fdc542c6bbec756dc4a9085c319fe6f0c9e1bb043f01a43c12aa6f4dff8b1220e7f16bc56dee9ccb59fb7c3b7aa6bb33b41c33d8e4b03b6b9cb89491504210dd691b46ffe2862387339d2b62a9dc4c20d629e23eb8b06490c4999433c1b4626fb4d21517072bd8e82511c115ee47bf9a5e40f0a74177f5b573db2e277459877a01b172e026cbb3f76aaf0c61f244584f3a76804dea62175a80d777238",
        "proof_hash": "660d706330fc36f611c50d90cb965fddf750cc91f8891a58b5e39b83a5fc6b46",
        "spent": false
      },
      {
        "block_height": 376151,
        "commit": "08ecd94ae293863286e99d37f4685f07369bc084ba74d5c59c7f15359a75c84c03",
        "merkle_proof": null,
        "mmr_index": 4107711,
        "output_type": "Coinbase",
        "proof": "7083884b5f64e4e61fb910d2c3c603f7c94490716d95e7144b4c927d0ca6ccc0e069cc285e25f38ee90c402ef26005cad2b4073eeba17f0ae3ea2b87095106ef00634f321d8a49c2feaad485bc9ee552564a6a883c99886d0d3a85af3490d718f5a5cbc70f9dcc9bf5d987fb6072132a4c247d4bbd4af927532a887b1e4250b7277771f6b82f43f4fb5a48089ed58e7d3190a19197e07acfed650f8b2cd5f103e994fb3d3735c5727f06f302bd1f182586297dd57a7951ff296bdf6106704abedc39db77f1293effaa7496a77d19420a6208bc1c589b33dad9540cb6180cccf5e085006b01309419f931e54531d770e5fe00eca584072692a7e4883fd65ed4a7c460665608ab96bf0c7d564fe96a341f14066db413a6fddc359eb11f6f962aca70ca1414c35d7941ce06b77d0a0606081b78d5e64a4501f8e8eba9f0e0889042bc54b4cbfd71087a95af63e0306dba214084d4860b0ce66dc80af44224e5a6fef55800650b05cf1639f81bfdc30950f3634d1fd4375d50c22c7f13f3dfb690e5f155a535aff041b7f800bfe74c60f606e8ab47df60754a0e08221c2a50abe643bb086433afd040a7e6290d1d00b3fe657be3bb05c67f90eb183c2acb53c81e1ca15cd8d35fe9d7d52d8f455398e905bdc77ffb211697d477af25704cf9896e8ce797f4fed03e2ba1615e3ad5646eecaa698470f99437d01d5193f041201502763e8bde51e6dc830b5c676d05c8f7f87c4972c578b8d9d5922ba29f6e4a89a123311d02b5ac44a7d5307f7ed5e4e66aaf749afc76c6fc1114445d6fafeea816a0f985eeacdbe9e6d32a8514ca4aaf7faad4e9d43cde55327ac84bac4d70a9319840e136e713aa31d639e43302f3c71a79f08f4e5c9a19a48d4b46403734cd8f3cc9b67bc26ea8e2a01e63a6f5be6e044e8ed5db5f26d15d25de75f672a79315c5e2407e",
        "proof_hash": "7cf77fdaecef6c6fc01edca744c1521581f854a9bac0153971edbb1618fc36ad",
        "spent": false
      },
      {
        "block_height": 376154,
        "commit": "095c12db5e57e4a1ead0870219bda4ebfb1419f6ab1501386b9dd8dc9811a8c5ff",
        "merkle_proof": "00000000003eadc6000000000000000e13c509a17cbb0d81634215cd2482ab6d9eb58b332fcbe6b2c4fa458a63d3cb0dfe3614ebe6e52657870df225d132179fa1ea0fdc2105f0e51d03bc3765a9cd059c60d434a7cae0a3d669b37588c25410f57405c841312cfa50cf514678877a3f4ce8bd3e57723ba75a2b7d61027b2088fbabebdb7336b97ea88b00a7e809a6245def980eba18d987601f4cbd6c3cc9f12a5684fe7a1bc2565a9f8ab63c2db1afa8304f5e23d4754cd97f29c8b06dcb3de4f6d3a83079676b6e9941afe5553a7195384b564ecd6d37522cb5e452cc930d2b549af22698a8fd9bf6cad05a06b09e3f6e672b94e82c0255394b5c187ab76fda653a2491378997ba3d49f9d9c34ca93bc627fe5d98b327c03d429b5473f62672e9d73c4eafd9cb8f62e5158a1ec7eb56653696b10fb9ec205f5e4d1c7a1f3e2dd2994b12eeed93e84776d8dcd8a5d78aecd4f96ae95c0b090d104adf2aa84f0a1fbd8d319fea5476d1a306b2800716e60b00115a5cca678617361c5a89660b4536c56254bc8dd7035d96f05de62b042d16acaeff57c111fdf243b859984063e3fcfdf40c4c4a52889706857a7c3e90e264f30f40cc87bd20e74689f14284bc5ea0a540950dfcc8d33c503477eb1c60",
        "mmr_index": 4107717,
        "output_type": "Coinbase",
        "proof": "073593bc475478f1e4b648ab261df3b0a6e5a58a617176dd0c8f5e0e1d58b012b40eb9b341d16ee22baf3645ea37705895e731dee5c220b58b0f780d781806a10dfa33e870d0494fba18aaa8a7a709bfb3ddf9eb3e4e75a525b382df68dc6f710275cdffb623373c47c1310ae63479826f435ca4520fdc13bb0d995b7d9a10a7587d61bd4a51c9e32c87f3eb6b0f862cdff19a9ac6cb04d6f7fafb8e94508a851dcf5dc6acea4271bb40117a45319da5522b966091b089698f4f940842458b5b49e212d846be35e0c2b98a00ac3d0b7ceaf081272dbed8abd84fe8f26d57bac1340e8184602436ed8c4470ef9dc214df3405de0e71703abec4456b15e122a94706852bb476213ceadf00529d00d8d3b16dc57f4e4a9a86dacfa719e00366728de42f3f830e73f6113f1e391fab07eba1b40f6466203b0ce14701230e934f6138c575660a03dbb0e59d7295df3115a4fc0909a5520d74657b319fc83481079ad6c13400175e39fa2b86071ba563ce8836320713ef8f55d4e90bee3f57df96c7aef0f2e896f57192fae9675471cd9751bcaf2b15e5a65a9733a6f7f9b8147b8f6e8dac51d056018d411fd252225cf88e56f143143f49e8a0d2e43c10de0442dbc84966817532b1256b6769db987526790a389c371a1fe7a36eacffef82877b4db7a9b5e58722ffbd0fc4fdbd7624365ee326bb8b1e60b999f513715b30f37ef6116eabf53b3524b46c33a1fac49205b39e24aa388d823269c1fc43c3599a06b69433a0a47a03bd871321afb7846a6dbfd5891bd84f89c556231745c929d08445f66f332857bfda1c4f86ae58a01007b7303f870ac24e0ba72d84c0ef4903ac2ff777e2c2dcb4d8e303c74e0c8a559686b4d4c25024ee97601787d4e5a97224af41e5d35d91744292f5a41f64d4e1cae77bebebd77a473f3b54e86f7221aac230942f0468",
        "proof_hash": "5dd69c083e2c0fd797a499bbafedee0728849afa3476034280ecadf6eb4bffc2",
        "spent": false
      },
      {
        "block_height": 376153,
        "commit": "0948cb346b7affe004a6f84fa4b5b44995830f1c332b03537df4c258d51d1afb50",
        "merkle_proof": "00000000003eadc4000000000000000dfe3614ebe6e52657870df225d132179fa1ea0fdc2105f0e51d03bc3765a9cd059c60d434a7cae0a3d669b37588c25410f57405c841312cfa50cf514678877a3f4ce8bd3e57723ba75a2b7d61027b2088fbabebdb7336b97ea88b00a7e809a6245def980eba18d987601f4cbd6c3cc9f12a5684fe7a1bc2565a9f8ab63c2db1afa8304f5e23d4754cd97f29c8b06dcb3de4f6d3a83079676b6e9941afe5553a7195384b564ecd6d37522cb5e452cc930d2b549af22698a8fd9bf6cad05a06b09e3f6e672b94e82c0255394b5c187ab76fda653a2491378997ba3d49f9d9c34ca93bc627fe5d98b327c03d429b5473f62672e9d73c4eafd9cb8f62e5158a1ec7eb56653696b10fb9ec205f5e4d1c7a1f3e2dd2994b12eeed93e84776d8dcd8a5d78aecd4f96ae95c0b090d104adf2aa84f0a1fbd8d319fea5476d1a306b2800716e60b00115a5cca678617361c5a89660b4536c56254bc8dd7035d96f05de62b042d16acaeff57c111fdf243b859984063e3fcfdf40c4c4a52889706857a7c3e90e264f30f40cc87bd20e74689f14284bc5ea0a540950dfcc8d33c503477eb1c60",
        "mmr_index": 4107716,
        "output_type": "Coinbase",
        "proof": "72950da23ad7f0d0381e2f788bf0ac6b6bcb17aaccf0373534122a95714d2d0dbf6a24822b4aab0711a595c80bc36122957111c39292f2a36a973252fb88cbda0b1d61ea8ea84f5171a61f751cac97332637b7cf74cc73144b912ba700dedaa60895f06e947f1e42a8c79d70f924f45fdcb6df5d30289f36ff77d0ae368df5775a739b7a25cbfb63f0cdbdc167b046067c2a021fe0950c7b67515b185b9e4a00ce63b795d49ae184fe5cc726d72fc05d717c4fb55dd5f65967dc282d3c47cb6f8a92cb696e5a1d8cca21214bc766e3de6271791cebf646cda97ae77035da16606f3397f71e103137358c97b9943c3e15403184f61230bd0e3954c7681a0891aa7a0cc32e82d830fb7d8759a04d1da7058630a853508df095142f22158c28bd5e3f2477ad6c8990e63d0377a0fa3d588b6584453778eb38cbaec8a33c1d3772c97a826d4a2f6953c35342993b04567e9fea6fc64fb714653f934faa1a8f635d39eb2903de4bed960a3df07dce7c2e3ff517bbc15f467d0190a579bc07b0f1a910b23269d794835bbb34e8318dcc4fd4159f8f03faa77842d445cf61af9e33caf46aa5fae0812a6476a09c0757e929271a96a245701ab14c1fdd836b92b7e763afa623017f68f1bc4eb716ce735820a1311b743dd8d5c6bb275a2e4e7d2eff8f45417b60cc937086c3e7fd3b612ae064d7237eb6a7bd1a39d8575fac312068fa060bc1ceac4df0754601edaf04ecb1b89c0661ea01a593c3763e456bebbd8487edc0ff3bc6f203965cd92b1706070c59a3795f9dee23087cea0aaec015f1b7bfe4df81818d7a37af781ca7b757ace2fa489f85215ecb85976b1c74c7f1df6d834a8bc63e887407ef6e233c55ea040bc5f2471e99ebc92f2283ff592ff751d9226bd105e68e187c91ecb236c9fa4fb060ae4d706c571ac2123da1debd12737d98be118578",
        "proof_hash": "0ce421970d13fe9b3981e308c5d0b549982cdda9f69918289cd95ffcd09e0fc2",
        "spent": false
      }
    ]
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

### get_peers

Retrieves information about peers. If `null` is provided, `get_peers` will list all stored peers.

```JSON
{
    "jsonrpc": "2.0",
    "method": "get_peer",
    "params": ["70.50.33.130:3414"],
    "id": 1
}
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": {
    "Ok": [
      {
        "addr": "70.50.33.130:3414",
        "ban_reason": "None",
        "capabilities": {
          "bits": 15
        },
        "flags": "Defunct",
        "last_banned": 0,
        "last_connected": 1570129317,
        "user_agent": "MW/Grin 2.0.0"
      }
    ]
  }
}
```

### get_connected_peers

Retrieves a list of all connected peers.

```JSON
{
    "jsonrpc": "2.0",
    "method": "get_peer",
    "params": [],
    "id": 1
}
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": {
    "Ok": [
      {
        "addr": "35.176.195.242:3414",
        "capabilities": {
          "bits": 15
        },
        "direction": "Outbound",
        "height": 374510,
        "total_difficulty": 1133954621205750,
        "user_agent": "MW/Grin 2.0.0",
        "version": 1
      },
      {
        "addr": "47.97.198.21:3414",
        "capabilities": {
          "bits": 15
        },
        "direction": "Outbound",
        "height": 374510,
        "total_difficulty": 1133954621205750,
        "user_agent": "MW/Grin 2.0.0",
        "version": 1
      },
      {
        "addr": "148.251.16.13:3414",
        "capabilities": {
          "bits": 15
        },
        "direction": "Outbound",
        "height": 374510,
        "total_difficulty": 1133954621205750,
        "user_agent": "MW/Grin 2.0.0",
        "version": 1
      },
      {
        "addr": "68.195.18.155:3414",
        "capabilities": {
          "bits": 15
        },
        "direction": "Outbound",
        "height": 374510,
        "total_difficulty": 1133954621205750,
        "user_agent": "MW/Grin 2.0.0",
        "version": 1
      },
      {
        "addr": "52.53.221.15:3414",
        "capabilities": {
          "bits": 15
        },
        "direction": "Outbound",
        "height": 0,
        "total_difficulty": 1133954621205750,
        "user_agent": "MW/Grin 2.0.0",
        "version": 1
      },
      {
        "addr": "109.74.202.16:3414",
        "capabilities": {
          "bits": 15
        },
        "direction": "Outbound",
        "height": 374510,
        "total_difficulty": 1133954621205750,
        "user_agent": "MW/Grin 2.0.0",
        "version": 1
      },
      {
        "addr": "121.43.183.180:3414",
        "capabilities": {
          "bits": 15
        },
        "direction": "Outbound",
        "height": 374510,
        "total_difficulty": 1133954621205750,
        "user_agent": "MW/Grin 2.0.0",
        "version": 1
      },
      {
        "addr": "35.157.247.209:23414",
        "capabilities": {
          "bits": 15
        },
        "direction": "Outbound",
        "height": 374510,
        "total_difficulty": 1133954621205750,
        "user_agent": "MW/Grin 2.0.0",
        "version": 1
      }
    ]
  }
}
```

#### ban_peer

Bans a specific peer.

```JSON
{
    "jsonrpc": "2.0",
    "method": "ban_peer",
    "params": ["70.50.33.130:3414"],
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

#### unban_peer

Unbans a specific peer.

```JSON
{
    "jsonrpc": "2.0",
    "method": "unban_peer",
    "params": ["70.50.33.130:3414"],
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
