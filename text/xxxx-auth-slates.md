
- Title: auth-slates
- Authors: [Marek Narozniak](https://mareknarozniak.com/)
- Start date: July 7, 2022
- RFC PR: Edit if merged: [mimblewimble/grin-rfcs#90](https://github.com/mimblewimble/grin-rfcs/pull/90)
- Tracking issue: [Edit if merged with link to tracking github issue]

---

## Summary
[summary]: #summary

We propose a protocol allowing to use slatepack-encoded signatures of authentication purposes. Unlike in case of the EVM-based blockchains where regular signature is used for authentication, our protocol works without revealing the slatepack address.

## Motivation
[motivation]: #motivation

GRIN digital cash project has managed to gather an online community of people spread around various online services, web boards and messenger apps. Apart from the [GRIN Forum](https://forum.grin.mw/) there is no official place of interaction. This is due to a general tendency to resist centralization which is very common among the GRIN community members. We have already witnessed birth of first GRIN-specific websites, such as [grinnode.live](https://grinnode.live/) and [Slatepacks Market](https://slatepacks.com/). It seems reasonable to assume there will be many more constructed in the future.

Assuming there will be more GRIN-related online communities and it is common for GRIN enthusiasts to have a GRIN wallet, it seems logical to create a feature allowing authentication via the wallet. Especially that such features are already common in other blockchains [ref needed]. But unlike other blockchains, the "GRIN way" of implementing such features should embrace on of the core GRIN values, which is privacy. In this RFC we propose a protocol that allows the wallet to act as an alternative of a password manager allowing GRIN users to authenticate to online services without revealing their slatepack addresses.

## Community-level explanation
[community-level-explanation]: #community-level-explanation

This feature will allow you to use GRIN wallet to register and login to online services that support it. In this section we will briefly describe the user experience we have in mind.

When you wish to register to some website.
1. You open the website registration form you have only single input `username` where you provide your desired username and you click `next` to submit the form.
2. Service will validate your input and inform you if the username is taken. If not, it will display a slatepack message related to your registration as well as textarea input to provide a response and a `sign-up` button.
3. In your wallet you run `grin-wallet register` command. The wallet asks you to paste the slatepack from the website.
4. After you submit the registration slatepack, wallet shows username and URL of the service and asks you if this information is valid. You may cancel the operation or proceed.
5. If you decide to proceed, wallet will provide a response slatepack which you copy.
6. You paste the response slatepack to registration form textarea and click `sign-up`.
7. The service informs you if the account was successfully created.

When you wish to login to a service, you proceed as follows.
1. You open the website you wish to authenticate. There is a textarea input and `sign-in` button.
2. In your wallet you run `grin-wallet login`.
3. Wallet displays list of accounts with usernames and URLs of the services. You select one you wish to use and submit.
4. Wallet will display a login slatepack which you copy.
5. In the website you paste the slatepack and click `sign-in`.

Your accounts are going to be stored locally but even if you lose your database, as long as you have your wallet recovery phrase and remember the URLs of the services you registered and your usernames your accounts can be recovered using `grin-wallet register --recover` command.

This change will not break anything previously included in the wallet. We propose it to be based on additional key derivation mechanisms.

## Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This section will provide a more technical description of how we propose the auth slates to work. We consider two parties, `Steven` for service and `Ursula` for the user.

### Registration protocol

Ursula wishes to create the account. She fills in the web form and clicks `next`. Steven generates a slatepack with the following data encoded.

```json
{
    "version_info": {
        "version": 1
    },
    "authentication": "registration",
    "data": {
        "username": "picked-username",
        "url": "url-of-the-service"
    }
}
```

Ursula copies the slatepack, runs `grin-wallet register` and pastes it in as the input. Ursula's wallet will generate a new ED25519 private key of the following form

```
key = H(H(seed || username || url) || username || url)
```

where `username` and `url` come from the registration slatepack and `seed` is the wallet decrypted seed key. From that it follows that only Ursula, as the party capable of decrypting the wallet seed is capable of creating this authentication key. The double hashing prevents someone from getting a wallet seed using []preimage attack](https://en.wikipedia.org/wiki/Preimage_attack).

Once key is generated, Ursula's wallet will generate following slatepack

```json
{
    "version_info": {
        "version": 1
    },
    "authentication": "registration-response",
    "data": {
        "username": "picked-username",
        "url": "url-of-the-service",
        "signature": "00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"
    }
}
```

which has `signature` field in addition being a digital signature of the login request.

Ursula submits this slatepack into the web form and clicks `sign-up`. The online service will

1. Check if signature is valid
2. Extract the key from the signature and save it own database along with the associated username
3. Inform Ursula of the successful registration

### Login protocol

Now Ursula wishes to login to Steven's online service. She starts by opening the website and sees the login form composed of the slatepack input textarea and `sign-in` button.

Ursula runs `grin-wallet login`, picks the account the wishes to use and approves. The exact way of achieving this depends on the implementation and is not relevant here. It could be an interactive UI or could be command line options. Once approved, wallet will display a slatepack representing following encoded slate.

```json
{
    "version_info": {
        "version": 1
    },
    "authentication": "login",
    "data": {
        "username": "picked-username",
        "url": "url-of-the-service",
        "timestamp": 1657270059,
        "signature": "00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"
    }
}
```

The slatepack contains current timestamp as a measure of preventing repetition attacks. Ursula proceeds by pasting it in the web form. Steven's website will

1. Extract the key from the signature, check if it figures in the database.
2. If the key does not figure in the database the authentication will fail. If it does figure associated to provided username it will check if signature matches.
3. If the signature does not match the authentication will fail. Otherwise it will succed and create a server-side session for Ursula's account.

### Account recovery protocol

It might occurr that user loses the wallet database. As long as the wallet seed is backed up, account can be recreated as the private signing keys for authentication are derived from the wallet seed and username and URLs. If user remembers the services and usernames wallet can always recreate the signing keys.

The minimal implementation of this feature would not require any database at all, user would need to provide username and URL to the wallet every time wishes to login. This would not require registration command and thus neither the recovery command.

## Drawbacks
[drawbacks]: #drawbacks

[TODO]

## Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

Alternatively one could login by signing the login information using the slatepack address key. The disadvantage of that is the slatepack address would become known and associated with the account. In certain scenarios that might be a desired feature.

## Prior art
[prior-art]: #prior-art

[TODO]

## Unresolved questions
[unresolved-questions]: #unresolved-questions

General review of the idea:

1. We should consider other forms of login. For instance, instead of deriving a key and providing a signature using this derived key we could use a zero-knowledge proof that owner of some wallet wishes to register to this particular service. This way no additional key is required as the zero-knowledge proof would prove there exists a signature without revealing the actual signature.
2. We should consider other way of deriving the authentication keys.
3. We should consider some common attack scenarios to evaluate the security aspects of this proposal.

Low-level:

1. Is double hashing sufficient to prevent getting someone else's wallet seed using the [preimage attack](https://en.wikipedia.org/wiki/Preimage_attack)?

## Future possibilities
[future-possibilities]: #future-possibilities

Implementing this proposal could result with improved privacy of GRIN users and stronger connection of the GRIN tech with the web. It could possibly motivate other GRIN-related projects for online communities.

In the future we could consider integrating zero-knowledge proof mechanism such as [zk-SNARK](https://www.investopedia.com/terms/z/zksnark.asp) to allow one to link online accounts. The current proposal makes it impossible to link users. In general this is considered an advantage, however in certain scenarios one might want to prove that same username in different services is claimed by the owner of the same wallet. A zero-knowledge proof such as zk-SNARK could prove that both keys are derived from same seed without revealing the seed.

## References
[references]: #references

- [Keybase discussion](keybase://chat/grincoin#cryptography/5744)
