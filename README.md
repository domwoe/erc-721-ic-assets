# Hosting ERC 721 metadata on the Internet Computer (IC)


## What are NFTs?

NFT is the short form of Non-fungible Tokens, i.e a unique representation of a digital asset. NFTs are mostly know for PFPs, e.g. the hexagonal profile pictures on Twitter, or POAPs to collect proofs of attending an event or finishing a course. 
Non-Fungible Tokens (NFTs) on Ethereum and other EVM-compatible platforms are usually represented following the [ERC-721 standard](https://eips.ethereum.org/EIPS/eip-721).

## Why using the Internet Computer?

In almost all cases, the actual asset, the image, music, or video, the NFT represents is not stored on-chain, because costs are prohibitive. Instead, 
the standard includes an optional metadata extension" which allows you smart contract to be interrogated for its name and for details about the assets which your NFTs represent". Typically the assets themselves are stored on IPFS using a pinning service like [Pinata](https://www.pinata.cloud/). 

You can host an NFT entirely on the Internet Computer, but since the IC is young and diverts from the common programming environment kickstarted by Ethereum, the ecosystem of tools, marketplaces, and wallets is still much smaller. However, you can benefit from the rich tooling of the EVM ecosystem, while still benefiting from some of the unique capabilities of the IC (though this tutorial only shows the basics).

- NFT metadata can be dynamic since the image can be generated dynamically in the canister/
- Dynamic data to affect NFT metadata can be fetched from off-chain sources via [HTTPS outcalls](https://internetcomputer.org/https-outcalls).
- The IC provides true cryptographic randomness as input for unique metadata.
- You can mint NFTs on Ethereum directly from a canister smart contract on the IC using [Threshold ECDSA](https://internetcomputer.org/docs/current/developer-docs/integrations/t-ecdsa/).

## How to host ERC 721 metadata on the Internet Computer

In this tutorial, we'll first host the NFT metadata using a canister smart contract on the Internet Computer. This will give us an URL that resolves to JSON metadata document which itself reloves to the actual asset. Both assets will be stored on the Internet Computer. Given the URL, we can mint an NFT on Ethereum 

You will learn how to deploy a canister smart contract on the Internet Computer that can respond to HTTP requests directly. We will do this using two different approaches

1) We'll use a predefined asset canister, that is shipped with dfx, a command line tool to interact with the Internet Computer. Thereby you'll learn about asset certification and the service worker.
2) We'll create a canister that implements the `http_request` method in Motoko. Thereby you'll see an example of a Motoko canister and you'll use the Motoko Playground to deploy a canister. (Though you shouldn't use the Playground to deploy a long-living canister, and rather use the project in `ic/erc_721_simple_canister`.)


### Prerequisites

This tutorial assumes that you know how to deploy an ERC-721 compatible NFT on an EVM chain, but you don't know about the Internet Computer. To get started with developing canisters on the Internet Computer you to install the dfx command line tool:

```
 sh -ci "$(curl -fsSL https://internetcomputer.org/install.sh)"
```

Furthermore, you need some cycles to pay for your canister smart contracts. You can get 20T of free cycles at https://facuet.dfinity.org. More information about cycles cost on the IC is [here](https://internetcomputer.org/docs/current/developer-docs/deploy/computation-and-storage-costs).

If you want to get some more info on the comparison between the Internet Computer and Ethereum, have a look at this blog post: [The Internet Computer for Ethereum Developers](https://medium.com/dfinity/the-internet-computer-for-ethereum-developers-3331b50db31b).

If you don't know how to deploy an NFT contract on Ethereum and mint an NFT, then either deploy everything on the Internet Computer ;) or head over to this three part series on ethereum.org
- [How to write and deploy an NFT](https://ethereum.org/en/developers/tutorials/how-to-write-and-deploy-an-nft/)
- [How to mint an NFT](https://ethereum.org/en/developers/tutorials/how-to-mint-an-nft/)
- [How to view your NFT in MetaMask](https://ethereum.org/en/developers/tutorials/how-to-view-nft-in-metamask/)

### Overview

`erc_721` contains the project to deploy and mint an ERC-721 compatible NFTs on Ethereum based on the tutorials mentioned above. If you don't want to follow the tutorials in detail, the only things you need are an Alchemy API key and an Ethereum account with some (testnet) ETH.

`ic` contains two different projects to host assets on the Internet Computer as will be explained next.

### Two ways to host 

Canister smart contracts on the Internet Computer can expose a `http_request` method to serve content directly via HTTPS. In order to serve the metadata, we can either create a custom canister that exposes the `http_request` method or use an asset canister that is shipped directly with dfx. In the asset canister, the assets are stored in a certified data structure, which allows to verify the authenticity of the assets, and the canister sets a special `IC-Certification` HTTP header that allows a client to verify the authenticity of the assets.

#### Using an asset canister to host NFT metadata

```
dfx new erc_721_asset_canister
cd erc_721_asset_canister
```
We clean up some up files and add an `image.png` and a `metadata.json` to the `assets` folder. The file structure should look as follows

```
├── README.md
├── dfx.json
└── src
    └── erc_721_asset_canister
        └── assets
            ├── image.png
            └── metadata.json
```

where `metadata.json` is a metadata document following the ERC-721 Metadata JSON Schema, e.g. 
```
{
    "name": "My IC asset NFT",
    "description": "An NFT hosted in an asset canister on the Internet Computer",
    "image": "https://<canisterId>.ic0.app/image.png"
}
```
and `image.png` the actual image that your NFT will represent.

In order to fill in the `<canisterId>`, we register a new canister on the Internet Computer.
```
dfx canister create --network=ic erc_721_asset_canister
```
You will see something like:
```
Creating canister erc_721_asset_canister...
erc_721_asset_canister canister created on network ic with canister id: eudls-gyaaa-aaaap-qauia-cai
```

Congratulations! You created a canister on the IC. It's still empty, but we can now fill in the `canisterId` in `metadata.json`.

Having done this, we can deploy the canister. This will install the WASM module of the asset canister and upload the assets.

```
dfx deploy --network=ic
```


The assets will be now served under the following URLs:`https://<canisterId>.ic0.app` and `https://<canisterId>.raw.ic0.app`. 

If you go to `https://<canisterId>.ic0.app/image.png`, you will see a service worker loading screen. This is because the [boundary nodes](https://internetcomputer.org/how-it-works/boundary-nodes/) of the Internet Computer first return a service worker which verifies the certification of the assets returned by the (asset) canister. You can read more about asset certification [here](https://internetcomputer.org/how-it-works/asset-certification/). If you use the `raw` URL, then the boundary nodes don't return the service worker and just translate your HTTP request into a proper call to the API of the Internet Computer. Since your browser does not know how to verify the certification, a single node machine of the Internet Computer, or a single boundary node could return fake data. 

Now that we have our asset canister deployed, we can mint an NFT that points to the metadata in our asset canister, following the tutorial above. 

#### Writing your own simple metadata canister in Motoko

Certification aside, we can implement a canister that serves the assets by implementing the `http_request` method ourselves. Here, we use the Motoko programming language.


Our canister implements the `http_request` method to serve the JSON metadata and the actual image. In the example, we directly embedded the image in the code as a base64 encoded string. You can see the complete example in the [Motoko Playground](https://m7sm4-2iaaa-aaaab-qabra-cai.raw.ic0.app/?tag=354553896).

When clicking on `Deploy` in the Motoko Playground, the canister gets deployed to the IC, and you should see something like the following in the log at the bottom: 
```
[12:56:10] Compiling code...
[12:56:11] Warning in file Main.mo:38:8 this pattern of type ?Text__8 does not cover value null
[12:56:11] Compiled Wasm size: 204KB
[12:56:11] Deploying code...
[12:56:11] Requesting a new canister id...
[12:56:20] Got canister id zydb5-siaaa-aaaab-qacba-cai
[12:56:26] Code installed at canister id zydb5-siaaa-aaaab-qacba-cai
```
If you now go to https://zydb5-siaaa-aaaab-qacba-cai.ic0.app/metadata in the browser, you should see the following

```
Body does not pass verification
```
This is due to the service worker. The service worker tries to verify the certificate, but we haven't implemented certification. Instead, if you go to https://zydb5-siaaa-aaaab-qacba-cai.raw.ic0.app/, you should get the JSON metadata:

```
{"title": "Asset Metadata","type": "object","properties": {"name": "My IC NFT ""description": "An NFT hosted on the Internet Computer""image": "https://zydb5-siaaa-aaaab-qacba-cai.raw.ic0.app/image.png", }}
```

and an image at https://zydb5-siaaa-aaaab-qacba-cai.raw.ic0.app/image.png.


### Deploying and Minting the NFT on Ethereum

We assume that you have already deployed your NFT contract following the tutorial [here](https://ethereum.org/en/developers/tutorials/how-to-write-and-deploy-an-nft/), and we can directly mint our NFT.

I deployed an ERC-721 NFT contract to the Ethereum Goerli Testnet at [0x96bAc7564C8Ab5b8baCA5e739FbB98e44dB53707](https://goerli.etherscan.io/address/0x96bAc7564C8Ab5b8baCA5e739FbB98e44dB53707).


The script to do this is `erc_721/scripts/mint-nft.js`. The only thing we need to do is to adapt the `METADATA_URL` to the proper URL
```
const METADATA_URL = "https://zydb5-siaaa-aaaab-qacba-cai.raw.ic0.app/metadata";
```
and run the script
```
node scripts/mint-nft.js
```

#### Checking out the NFT on OpenSea

You can now check your NFT on OpenSea, since I deployed my contract on the Goerli Testnet, I need to go to `https://testnets.opensea.io/` and insert the contract id into the search bar. This should lead you to your NFT. See mine [here](https://testnets.opensea.io/assets?search[query]=0x96bAc7564C8Ab5b8baCA5e739FbB98e44dB53707).

## Important Remarks

### Cycles

Hosting on the Internet Computer is relatively cheap. Currently, you have to pay around USD 5/GB/year, but there's currently no possibility for permanent hosting. The Internet Computer implements what's called the [Reverse Gas Model](https://internetcomputer.org/features/reverse-gas/), this allows easy onboarding of new users since users don't have to pay tokens to interact with canister smart contracts. In contrast, canister smart contracts have to pay for their resource usage in cycles, and if they run out of cycles they get de-allocated, i.e. code and state get removed, and only the metadata is retained. To prevent this from happening suddenly, the Internet Computer has the concept of a freezing threshold. The freezing threshold is defined in seconds (and corresponds to approximately 30 days by default) and is converted into a threshold in cycles internally. If the canister balance touches this threshold, the canister is frozen and won't respond to a call anymore, but the balance will keep it alive for the time given by the threshold (i.e. 30 days by default). An NFT asset canister should define a higher threshold of a few months or even years, to be sure that the NFT owner has enough time to notice and top up the canister. 

If you're a controller of a canister, you can check the balance with 
```
dfx canister <canister name or principal> status
```
Alternatively, you can get the balance inside the canister and expose it via an API or HTTP endpoint

```
import Cycles "mo:base/ExperimentalCycles";

let balance = Cycles.balance())
```
or use a tool like [Canistergeek](https://cusyh-iyaaa-aaaah-qcpba-cai.raw.ic0.app/).

### Controllers

Canister smart contracts are upgradable by default. Principals, the name of identifiers on the IC, that are registered as controllers for a canister, can upgrade, stop, or delete the canister at any time. for many NFT projects it is important that the NFT assets are immutable, hence the list of controllers should be empty or only include a verified canister, with properly defined rights such as the [blackhole canister](https://github.com/ninegua/ic-blackhole), that allows getting status information, such as the balance, from a canister.

### The ic0.app domain

One of the risks with hosting NFT metadata on the IC is that the link from the ERC-721 contract on Ethereum to the IC is via the Domain Name System (DNS). The .app TLD is owned by Google and the ic0 subdomain is owned by DFINITY, but can in principle be taken away. If this happens, the NFT metadata can't be resolved anymore. Similar to the `ipfs://<cid>` URI scheme, a permanent addressing scheme for resources on the Internet Computer is needed. This could be based on the [Decentralized Identifier (DID)](https://www.w3.org/TR/did-core/) specification or a new URI scheme like `icp://<canisterId>/`.



