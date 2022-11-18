# Minimal example of an asset canister to host ERC-721 NFT metadata


## Running the project locally

If you want to test your project locally, you can use the following commands:

```bash
# Starts the replica, running in the background
dfx start --background

# Deploys your canisters to the replica and generates your candid interface
dfx deploy
```
Once the job completes, your application will be available at `http://localhost:4943?canisterId={asset_canister_id}`.



