# Becoming a Validator

::: danger
If you intend to become a `MainNet` Validator, it is **HIGHLY** recommended that you practice on `TestNet` first 
in order to fully familiarise yourself with the process.
:::

::: warning IMPORTANT
Whenever you use `und` to send Txs or query the chain ensure you pass the correct data to the `--chain-id` and if 
necessary `--node=` flags so that you connect to the correct network!
:::

::: tip
Validator nodes require high availability and uptimes. The following guide therefore assumes the node is running on a 
server/cloud VM, as opposed to a laptop or home PC.
:::

#### Contents

[[toc]]

## Prerequisites

Before continuing, ensure you have gone through the following docs:

1. [Installing the software](../software/installation.md)
2. [Join a Public Network](join-network.md)
3. [Accounts and Wallets](../software/accounts-wallets.md)
4. Chain ID - if you haven't already, you can get the current chain ID by running:

```
jq --raw-output '.chain_id' $HOME/.und_mainchain/config/genesis.json
```

**You will need to run this on the same host that is running your validator node**

The above command assumes you have downloaded the appropriate genesis for the network you wish to become a Validator 
on to the default `$HOME/.und_mainchain` directory.

::: warning IMPORTANT
you will need an account with sufficient FUND to self-delegate to your validator node. For **TestNet** nodes,
you can use the [TestNet Faucet](https://faucet-testnet.unification.io).
:::

::: tip
if you intend to fully participate in the running of a network, your node will need to be permanently available and 
online. In which case, you will need to investigate running `und` as a [background service](run-und-as-service.md)
:::

## Creating a validator

::: danger IMPORTANT!
keep your `$HOME/.und_mainchain/config/node_key.json` and `$HOME/.und_mainchain/config/priv_validator_key.json` files 
**safe and secure**! These are required for your node to propose and sign blocks. If you ever migrate your node to a 
different host machine or need to restore your node, you will need these!
:::

::: tip NOTE
The following commands assume you have initialised your node using the default `$HOME/.und_mainchain` directory. If
you are using a different location, use the `--home` flag.
:::

**If required, SSH to the host running your validator node - the public key output by the following command must be the
public key of your validator node!**

The first thing you will need is your node's Tendermint validator public key. This will be used to register your node 
as a Validator on the network. To get the key, run:

```bash
und tendermint show-validator
```
This will output your node's public key. Make a note of it, as it will be required soon. You will need the entire 
JSON string

::: warning IMPORTANT
Before continuing, ensure your node has fully synced with the network and downloaded all the blocks (this may take 
a while, so go and make a brew). You can check the status of your node's sync by running the following from the node 
host:

```bash
curl -s http://localhost:26657/status | jq '.result.sync_info.catching_up'
```

if the value is `false`, the node is fully synced
:::

To create your Validator, you will need to generate, sign and broadcast a special transaction to the network which will 
register your Tendermint validator public key and stake the amount of FUND (in `nund`) specified (via self-delegation). 
Run the following command, modifying as required (**this command can be run from your local host, or the node host,
depending on where you generated the account wallet in Prerequisite #3**)

```bash
und tx staking create-validator \
  --amount=STAKE_IN_NUND \
  --pubkey=NODE_TENDERMINT_PUBLIC_KEY \
  --moniker="YOUR_VALIDATOR_MONIKER" \
  --website="YOUR_WEBSITE_URL" \
  --identity=16_DIGIT_KEYBASE_IO_ID \
  --details="NODE_DESCRIPTION" \
  --security-contact="SECURITY_CONTACT_EMAIL" \
  --chain-id=CHAIN_ID \
  --commission-rate="0.10" \
  --commission-max-rate="0.20" \
  --commission-max-change-rate="0.01" \
  --min-self-delegation="1" \
  --gas="auto" \
  --gas-prices="0.25nund" \
  --gas-adjustment=1.5 \
  --from=SELF_DELEGATOR_ACCOUNT
```

**Mandatory fields**

`STAKE_IN_NUND`: the amount in nund you want to delegate to yourself. For example, if you want to stake 1000 FUND, 
enter 1000000000000nund.

::: tip
You can use the built in `und` conversion tool to calculate this:

```
und convert 1000 fund nund.
```
:::

::: warning IMPORTANT
do not attempt to stake more than you have in your account. **Ensure you have enough FUND to pay for the transaction 
fees, and enough left over for future transactions such as withdrawing rewards!**
:::

`NODE_TENDERMINT_PUBLIC_KEY`: Your node's Protobuf JSON encoded tendermint public key, obtained earlier via the 
`und tendermint show-validator` command.

`CHAIN_ID`: the chain you are creating a validator for. This was obtained previously via the `jq` command, and will 
be for example `FUND-TestNet-2` or `FUND-MainNet-2` etc.

`SELF_DELEGATOR_ACCOUNT`: the name of the account being used to stake self-delegated FUND and sign the 
transaction — for example, the identifier you entered when running the `und keys add` command to 
create/import an account.

`YOUR_VALIDATOR_MONIKER`: a moniker which will publicly identify your Validator node on the network.

**Optional fields**

::: tip
Ensure you create your validator with as much of the following additional information as you can. It will be 
publicly visible, and help potential stakers connect with you
:::

`YOUR_WEBSITE_URL`: the URL for the site promoting your validation node

`16_DIGIT_KEYBASE_IO_ID`: Your 16 digit public [keybase.io](https://keybase.io) PGP public key ID if you have one 
and want to associate your ID to your validator node.

`NODE_DESCRIPTION`: a brief description of your validator node

`SECURITY_CONTACT_EMAIL`: Email address for the security contact for your validator node

**Commission Rates**

Your commission rates can be set using the `--commission-rate` , `--commission-max-change-rate` and 
`--commission-max-rate` flags.

`--commission-rate`: The % commission you will earn from delegators’ rewards. Keeping this relatively low can 
attract more delegators to your node.

`--commission-max-rate`: The maximum you will ever increase your commission rate to — you cannot raise commission 
above this value. Again, keeping this low can attract more delegators.

`--commission-max-change-rate`: The maximum you can increase the commission-rate by per day. For example, if your 
maximum change rate is 0.01, you can only make changes in 0.01 increments, so from 0.10 (10%) to 0.11 (11%).

::: warning IMPORTANT
The values for `--commission-max-change-rate` and `--commission-max-rate` flags **cannot be changed** after 
the `create-validator` command has been run.
:::

Finally, the `--min-self-delegation` flag is the minimum amount of `nund` you are required to keep self-delegated 
to your validator, meaning you must always have _at least_ this amount self-delegated to your node.

**Example: creating a TestNet validator**

```bash
und tx staking create-validator \
--amount=1000000000000nund \
--pubkey='{"@type":"/cosmos.crypto.ed25519.PubKey","key":"abc123...="}' \
--moniker="MyAwesomeNode" \
--website="https://my-node-site.com" \
--details="My node is awesome" \
--security-contact="security@my-node-site.com" \
--chain-id=FUND-TestNet-2 \
--commission-rate="0.07" \
--commission-max-rate="0.20" \
--commission-max-change-rate="0.01" \
--min-self-delegation="1000000000" \
--gas="auto" \
--gas-prices="0.25nund" \
--gas-adjustment=1.5 \
--from=my_new_wallet
```

The command will return a Tx hash, which you can use to query whether or not the transaction was successful:

```bash
und query tx TX_HASH --chain-id FUND-TestNet-2
```

::: tip
you can set the `--broadcast-mode` flag in the command to `block`. This will tell `und` to wait for the 
transaction to be processed in a block before returning the result. This will take up to 5-6 seconds to complete, 
but the Tx result will be included in the output.
:::

### Verify

You can verify your node is registered as a validator by running:

```bash
und query staking validator \
  $(und keys show SELF_DELEGATOR_ACCOUNT --bech=val -a) \
  --chain-id=CHAIN_ID
```

replacing `SELF_DELEGATOR_ACCOUNT` and `CHAIN_ID` accordingly.
