# ERC-721 Metadata Canister

Simple canister in Motoko to serve ERC-721 compatible metadata.

## Running the project locally

If you want to test your project locally, you can use the following commands:

```bash
# Starts the replica, running in the background
dfx start --background

# Deploys your canisters to the replica and generates your candid interface
dfx deploy
```

## Deploying to the Internet Computer

Make sure you've set up a wallet canister with some cycles, then just run

```
dfx deploy --network=ic
```