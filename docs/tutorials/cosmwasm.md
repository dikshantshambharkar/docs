---
sidebar_position: 6
sidebar_label: CosmWasm tutorial
description: Build a sovereign rollup with CosmWasm, Celestia, and Rollkit.
---

# 🗞️ CosmWasm rollup

:::tip difficulty
Intermediate
:::

CosmWasm is a smart contracting platform built for the Cosmos
ecosystem by making use of [WebAssembly](https://webassembly.org) (Wasm)
to build smart contracts for Cosmos-SDK. In this tutorial, we will be
exploring how to integrate CosmWasm with Celestia's
[data availability layer](https://docs.celestia.org/concepts/how-celestia-works/data-availability-layer)
using Rollkit.

:::tip note
This tutorial will explore developing with Rollkit,
which is still in Alpha stage. If you run into bugs, please write a Github
[Issue ticket](https://github.com/rollkit/docs/issues/new)
or let us know in our [Telegram](https://t.me/rollkit).
:::

:::danger caution

The script for this tutorial is built for Celestia's
[Mocha testnet](https://docs.celestia.org/nodes/mocha-testnet).
If you choose to use Arabica devnet,
you will need to modify the script manually.

:::

You can learn more about CosmWasm [here](https://docs.cosmwasm.com/docs/1.0).

In this tutorial, we will going over the following:

1. [Setting up your dependencies for your CosmWasm smart contracts](#setting-up-your-environment-for-cosmwasm-on-celestia)
2. [Setting up Rollkit on CosmWasm](#wasmd-installation)
3. [Instantiate a local network for your CosmWasm chain connected to Celestia](#contract-interaction-on-cosmwasm-with-celestia)
4. [Deploying a Rust smart contract to CosmWasm chain](#contract-deployment-on-cosmwasm-with-rollkit)
5. [Interacting with the smart contract](#contract-interaction-on-cosmwasm-with-celestia)

The smart contract we will use for this tutorial is one provided by
the CosmWasm team for Nameservice purchasing.

You can check out the contract [here](https://github.com/InterWasm/cw-contracts/tree/main/contracts/nameservice).

How to write the Rust smart contract for Nameservice is outside the scope of
this tutorial. In the future we will add more tutorials for writing CosmWasm
smart contracts for Celestia.

## 💻 CosmWasm dependency installations

### 🛠️ Environment setup

For this tutorial, we will be using `curl` and `jq` as helpful
tools. You can follow the guide on installing them
[here](https://docs.celestia.org/nodes/environment/#-install-wget-and-jq).

### 🏃 Golang dependency

The Golang version used for this tutorial is v1.18+

You can install Golang
by following our tutorial [here](https://docs.celestia.org/nodes/environment#install-golang).

### 🦀 Rust installation

#### 🔨 Rustup

First, before installing Rust, you would need to install `rustup`.

On Mac and Linux systems, here are the commands for installing it:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

:::danger caution

You will see a note similar to below after installing Rust:

```bash
Rust is installed now. Great!

To get started you may need to restart your current shell.
This would reload your PATH environment variable to include
Cargo's bin directory ($HOME/.cargo/bin).

To configure your current shell, run:
source "$HOME/.cargo/env"
```

If you don't follow the guidance, you won't be able to continue with the
tutorial!

:::

After installation, follow the commands here to setup Rust.

```bash
rustup default stable
cargo version

rustup target list --installed
rustup target add wasm32-unknown-unknown
```

Your output should look similar to below:

```bash
info: using existing install for 'stable-aarch64-apple-darwin'
info: default toolchain set to 'stable-aarch64-apple-darwin'

  stable-aarch64-apple-darwin unchanged - rustc 1.65.0 (897e37553 2022-11-02)
  
cargo 1.65.0 (4bc8f24d3 2022-10-20)
aarch64-apple-darwin
info: downloading component 'rust-std' for 'wasm32-unknown-unknown'
info: installing component 'rust-std' for 'wasm32-unknown-unknown'
```

### 🐳 Docker installation

We will be using Docker later in this tutorial for compiling a smart contract
to use a small footprint. We recommend installing Docker on your machine.

Examples on how to install it on Linux are found [here](https://docs.docker.com/engine/install/ubuntu).
Find the right instructions specific for
[your OS here](https://docs.docker.com/engine/install).

### 💻 Wasmd installation

Here, we are going to pull down the `wasmd` repository and replace Tendermint
with Rollkit. Rollkit is a drop-in replacement for Tendermint that allows
Cosmos-SDK applications to connect to Celestia's data availability network.

```bash
git clone https://github.com/CosmWasm/wasmd.git
cd wasmd
git fetch --tags
git checkout v0.27.0
go mod edit -replace github.com/cosmos/cosmos-sdk=github.com/rollkit/cosmos-sdk@v0.45.10-rollkit-v0.6.0-no-fraud-proofs
go mod edit -replace github.com/tendermint/tendermint=github.com/celestiaorg/tendermint@v0.34.22-0.20221202214355-3605c597500d
go mod tidy -compat=1.17
go mod download
make install
```

### ✨ Celestia node

You will need a light node running with test tokens on
[Mocha testnet](https://docs.celestia.org/nodes/mocha-testnet) in order
to complete this tutorial. Please complete the tutorial
[here](https://docs.celestia.org/developers/node-tutorial),
or start up yournode.

## 🌌 Setting up your environment for CosmWasm on Celestia

Now the `wasmd` binary is built, we need to setup a local network
that communicates between `wasmd` and Rollkit.

### 🗞️ Initializing CosmWasm rollup with a bash script

We have a handy `init.sh` found in this repo
[here](https://github.com/rollkit/docs/tree/main/docs/scripts/cosmwasm).

We can copy it over to our directory with the following commands:

<!-- markdownlint-disable MD013 -->
```bash
# From inside the `wasmd` directory
wget https://raw.githubusercontent.com/rollkit/docs/main/docs/scripts/cosmwasm/init.sh
```
<!-- markdownlint-enable MD013 -->

This copies over our `init.sh` script to initialize our
CosmWasm rollup.

You can view the contents of the script to see how we
initialize the CosmWasm Rollup.

:::danger caution

If you are on macOS, you will need to install md5sha1sum before starting your
rollup:

```bash
brew install md5sha1sum
```

:::

You can initialize the script with the following command:

```bash
bash init.sh
```

With that, we have kickstarted our `wasmd` network!

### 💠 Optional: see what's inside the script

You can skip this section, but it is important to know
how Rollkit is initializing the cosmwasm rollup.

Here are the contents of the script:

<!-- markdownlint-disable MD013 -->
```bash
#!/bin/sh

VALIDATOR_NAME=validator1
CHAIN_ID=celeswasm
KEY_NAME=celeswasm-key
TOKEN_AMOUNT="10000000000000000000000000uwasm"
STAKING_AMOUNT=1000000000uwasm
CHAINFLAG="--chain-id ${CHAIN_ID}"
TXFLAG="--chain-id ${CHAIN_ID} --gas-prices 0uwasm --gas auto --gas-adjustment 1.3"
NODEIP="--node http://127.0.0.1:26657"

NAMESPACE_ID=$(echo $RANDOM | md5sum | head -c 16; echo;)
echo $NAMESPACE_ID
DA_BLOCK_HEIGHT=$(curl https://rpc-mocha.pops.one/block | jq -r '.result.block.header.height')
echo $DA_BLOCK_HEIGHT

rm -rf "$HOME"/.wasmd
wasmd tendermint unsafe-reset-all
wasmd init $VALIDATOR_NAME --chain-id $CHAIN_ID

sed -i'' -e 's/^minimum-gas-prices *= .*/minimum-gas-prices = "0uwasm"/' "$HOME"/.wasmd/config/app.toml
sed -i'' -e '/\[api\]/,+3 s/enable *= .*/enable = true/' "$HOME"/.wasmd/config/app.toml
sed -i'' -e "s/^chain-id *= .*/chain-id = \"$CHAIN_ID\"/" "$HOME"/.wasmd/config/client.toml
sed -i'' -e '/\[rpc\]/,+3 s/laddr *= .*/laddr = "tcp:\/\/0.0.0.0:26657"/' "$HOME"/.wasmd/config/config.toml
sed -i'' -e 's/"time_iota_ms": "1000"/"time_iota_ms": "10"/' "$HOME"/.wasmd/config/genesis.json
sed -i'' -e 's/bond_denom": ".*"/bond_denom": "uwasm"/' "$HOME"/.wasmd/config/genesis.json
sed -i'' -e 's/mint_denom": ".*"/mint_denom": "uwasm"/' "$HOME"/.wasmd/config/genesis.json

wasmd keys add $KEY_NAME --keyring-backend test
wasmd add-genesis-account $KEY_NAME $TOKEN_AMOUNT --keyring-backend test
wasmd gentx $KEY_NAME $STAKING_AMOUNT --chain-id $CHAIN_ID --keyring-backend test
wasmd start --rollkit.aggregator true --rollkit.da_layer celestia --rollkit.da_config='{"base_url":"http://localhost:26659","timeout":60000000000,"fee":6000,"gas_limit":6000000}' --rollkit.namespace_id $NAMESPACE_ID --rollkit.da_start_height $DA_BLOCK_HEIGHT
```
<!-- markdownlint-enable MD010 -->

## 📒 Contract deployment on CosmWasm with Rollkit

### 🤖 Compile the smart contract

In a new terminal instance, we will run the following commands to pull down the
Nameservice smart contract and compile it:

```bash
git clone https://github.com/InterWasm/cw-contracts
cd cw-contracts
cd contracts/nameservice
cargo wasm
```

The compiled contract is outputted to:
`target/wasm32-unknown-unknown/release/cw_nameservice.wasm`.

### 🧪 Unit tests

If we want to run tests, we can do so with the following command in the
`~/cw-contracts/contracts/nameservice` directory:

```bash
cargo unit-test
```

### 🏎️ Optimized smart contract

Because we are deploying the compiled smart contract to `wasmd`,
we want it to be as small as possible.

The CosmWasm team provides a tool called `rust-optimizer` which we need
[Docker](#docker-installation) for in order to compile.

````mdx-code-block
import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

<Tabs groupId="network">
<TabItem value="amd" label="AMD">


### AMD Machines

Run the following command in the `~/cw-contracts/contracts/nameservice`
directory:

```bash
sudo docker run --rm -v "$(pwd)":/code \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/code/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  cosmwasm/rust-optimizer:0.12.6
```

This will place the optimized Wasm bytecode at `artifacts/cw_nameservice.wasm`.

</TabItem>
<TabItem value="arm" label="ARM">

### ARM Machines

Run the following command in the `~/cw-contracts/contracts/nameservice`
directory:

```bash
sudo docker run --platform linux/arm64 --rm -v "$(pwd)":/code \
  --mount type=volume,source="$(basename "$(pwd)")_cache",target=/code/target \
  --mount type=volume,source=registry_cache,target=/usr/local/cargo/registry \
  cosmwasm/rust-optimizer-arm64:0.12.8
```

This will place the optimized Wasm bytecode at `artifacts/cw_nameservice-aarch64.wasm`.

</TabItem>
</Tabs>
````

### 🚀 Contract deployment

Let's now deploy our smart contract!

````mdx-code-block
<Tabs groupId="network">
<TabItem value="amd" label="AMD">


### AMD Machines

Run the following in the `~/cw-contracts/contracts/nameservice` directory:

```bash
TX_HASH=$(wasmd tx wasm store artifacts/cw_nameservice.wasm --from celeswasm-key --keyring-backend test --chain-id celeswasm --gas-prices 0uwasm --gas auto --gas-adjustment 1.3 --node http://127.0.0.1:26657 --output json -y | jq -r '.txhash') && echo $TX_HASH
```

</TabItem>
<TabItem value="arm" label="ARM">

### ARM Machines

Run the following in the `~/cw-contracts/contracts/nameservice` directory:

```bash
TX_HASH=$(wasmd tx wasm store artifacts/cw_nameservice-aarch64.wasm --from celeswasm-key --keyring-backend test --chain-id celeswasm --gas-prices 0uwasm --gas auto --gas-adjustment 1.3 --node http://127.0.0.1:26657 --output json -y | jq -r '.txhash') && echo $TX_HASH
```

</TabItem>
</Tabs>
````

This will get you the transaction hash for the smart contract deployment. Given
we are using Rollkit, there will be a delay on the transaction being included
due to Rollkit waiting on Celestia's data availability layer to confirm the block
has been included before submitting a new block.

:::danger important
If you run into errors with variables on the previous command,
or commands in the remainder of the tutorial, cross-reference
the variables in the command with the variables in the `init.sh` script.
:::

## 🌟 Contract interaction on CosmWasm with Celestia
<!-- markdownlint-disable MD013 -->

In the previous steps, we have stored out contract's tx hash in an
environment variable for later use.

Because of the longer time periods of submitting transactions via Rollkit
due to waiting on Celestia's data availability layer to confirm block inclusion,
we will need to query our  tx hash directly to get information about it.

### 🔎 Contract querying

Let's start by querying our transaction hash for its code ID:

```bash
CODE_ID=$(wasmd query tx --type=hash $TX_HASH --chain-id celeswasm --node http://127.0.0.1:26657 --output json | jq -r '.logs[0].events[-1].attributes[0].value')
echo $CODE_ID
```

This will give us back the Code ID of the deployed contract.

In our case, since it's the first contract deployed on our local network,
the value is `1`.

Now, we can take a look at the contracts instantiated by this Code ID:

```bash
wasmd query wasm list-contract-by-code $CODE_ID --chain-id celeswasm --node http://127.0.0.1:26657 --output json
```

We get the following output:

```json
{"contracts":[],"pagination":{"next_key":null,"total":"0"}}
```

### 📃 Contract instantiation

We start instantiating the contract by writing up the following `INIT` message
for nameservice contract. Here, we are specifying that `purchase_price` of a name
is `100uwasm` and `transfer_price` is `999uwasm`.

```bash
INIT='{"purchase_price":{"amount":"100","denom":"uwasm"},"transfer_price":{"amount":"999","denom":"uwasm"}}'
wasmd tx wasm instantiate $CODE_ID "$INIT" --from celeswasm-key --keyring-backend test --label "name service" --chain-id celeswasm --gas-prices 0uwasm --gas auto --gas-adjustment 1.3 -y --no-admin --node http://127.0.0.1:26657
```

### 📄 Contract interaction

Now that we instantiated it, we can interact further with the contract:

```bash
wasmd query wasm list-contract-by-code $CODE_ID --chain-id celeswasm --output json --node http://127.0.0.1:26657
CONTRACT=$(wasmd query wasm list-contract-by-code $CODE_ID --chain-id celeswasm --output json --node http://127.0.0.1:26657 | jq -r '.contracts[-1]')
echo $CONTRACT

wasmd query wasm contract --node http://127.0.0.1:26657 $CONTRACT --chain-id celeswasm
wasmd query bank balances --node http://127.0.0.1:26657 $CONTRACT --chain-id celeswasm
```

This allows us to see the contract address, contract details, and
bank balances.

Your output will look similar to below:

```bash
{"contracts":["wasm14hj2tavq8fpesdwxxcu44rty3hh90vhujrvcmstl4zr3txmfvw9s0phg4d"],"pagination":{"next_key":null,"total":"0"}}
wasm14hj2tavq8fpesdwxxcu44rty3hh90vhujrvcmstl4zr3txmfvw9s0phg4d
address: wasm14hj2tavq8fpesdwxxcu44rty3hh90vhujrvcmstl4zr3txmfvw9s0phg4d
contract_info:
  admin: ""
  code_id: "1"
  created: null
  creator: wasm1y9ceqvnsnm9xtcdmhrjvv4rslgwfzmrzky2c5z
  extension: null
  ibc_port_id: ""
  label: name service
balances: []
pagination:
  next_key: null
  total: "0"
```

Now, let's register a name to the contract for our wallet address:

```bash
REGISTER='{"register":{"name":"fred"}}'
wasmd tx wasm execute $CONTRACT "$REGISTER" --amount 100uwasm --from celeswasm-key --chain-id celeswasm --gas-prices 0uwasm --gas auto --gas-adjustment 1.3 --node http://127.0.0.1:26657 --keyring-backend test -y
```

Your output will look similar to below:

```bash
DEIP --keyring-backend test -y
gas estimate: 167533
code: 0
codespace: ""
data: ""
events: []
gas_used: "0"
gas_wanted: "0"
height: "0"
info: ""
logs: []
raw_log: '[]'
timestamp: ""
tx: null
txhash: C147257485B72E7FFA5FDB943C94CE951A37817554339586FFD645AD2AA397C3
```

If you try to register the same name again, you'll see an expected error:

```bash
Error: rpc error: code = Unknown desc = rpc error: code = Unknown desc = failed to execute message; message index: 0: Name has been taken (name fred): execute wasm contract failed [CosmWasm/wasmd/x/wasm/keeper/keeper.go:364] With gas wanted: '0' and gas used: '123809' : unknown request
```

Next, query the owner of the name record:

```bash
NAME_QUERY='{"resolve_record": {"name": "fred"}}'
wasmd query wasm contract-state smart $CONTRACT "$NAME_QUERY" --chain-id celeswasm --node http://127.0.0.1:26657 --output json
```

You'll see the owner's address in a JSON response:

```bash
{"data":{"address":"wasm1y9ceqvnsnm9xtcdmhrjvv4rslgwfzmrzky2c5z"}}
```

With that, we have instantiated and interacted with the CosmWasm nameservice
smart contract using Celestia!
