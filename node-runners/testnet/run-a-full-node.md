# Run A Full Node

## How To Join Secret Network as a Full Node on Testnet <a href="#how-to-join-secret-network-as-a-full-node-on-testnet" id="how-to-join-secret-network-as-a-full-node-on-testnet"></a>

This document details how to join the Secret Network `testnet` as a full node. Once your full node is running, you can turn it into a validator in the optional last step.

### Requirements <a href="#requirements" id="requirements"></a>

{% hint style="danger" %}
Secret Network has strict Hardware Requirements. If your machine does not meet them, it will \*NOT\* work as a node.
{% endhint %}

* Ubuntu/Debian host (with ZFS or LVM to be able to add more storage easily)
* A public IP address
* Open ports `TCP 26656 & 26657` _Note: If you're behind a router or firewall then you'll need to port forward on the network device._
* Reading [Tendermint: Running in production](https://docs.tendermint.com/v0.34/tendermint-core/running-in-production.html)
* RPC address of an already active node. You can use `bootstrap.secrettestnet.io:26657`, or any other node that exposes RPC services. Alternate RPC nodes available in the [API Registry.](../../development/connecting-to-the-network.md)
* [Install SGX](install-sgx.md)

#### Minimum Requirements <a href="#minimum-requirements" id="minimum-requirements"></a>

* 16GB RAM
* 150GB SSD for Prune Everything, or default pruning. 1TB premium SSD for Archive nodes.
* 1 dedicated core of any Intel Skylake processor (Intel® 6th generation) or better (Xeon gen3 (Ice Lake) NOT supported)
* Motherboard with support for SGX in the BIOS

#### Recommended Requirements <a href="#recommended-requirements" id="recommended-requirements"></a>

* 32GB RAM
* 512GB SSD
* 2 dedicated cores of any Intel Skylake processor (Intel® 6th generation) or better (Xeon gen3 (Ice Lake) NOT supported)
* Motherboard with support for SGX in the BIOS

## Installation <a href="#installation" id="installation"></a>

### **Install SGX and `secretd`**

{% hint style="danger" %}
This guide assumes you've already installed the latest version of secretd and SGX. To setup an archive node, you must follow the [Archive Nodes](../archive-nodes.md) instructions.
{% endhint %}

For more information on SGX, see instructions for [SGX Installation](../node-setup/install-sgx.md) and [Verifying SGX](../misc/verify-sgx.md). See [Node Registration Information](../misc/registration-information.md) if you'd like a more comprehensive overview on what's happening in these steps.

### **Initialize Secret Network Configs**

Choose a **moniker** for yourself, and replace `<MONIKER>` with your moniker below. This moniker will serve as your public nickname in the network.

```bash
secretd init <MONIKER> --chain-id pulsar-2
```

This will generate the following files in `~/.secretd/config/`

* `genesis.json`
* `node_key.json`
* `priv_validator_key.json`

### **Download `genesis.json`**

The genesis file is how other nodes on the network know what network you should be on.

```bash
wget -O ~/.secretd/config/genesis.json "https://storage.googleapis.com/stakeordie-pulsar-2/genesis.json"
# verify genesis.json checksum
echo "a48a5c2ba3f0d0ee077fc9a24514caaed3914e23e0de7b88163bb4d25e0866b8 $HOME/.secretd/config/genesis.json" | sha256sum --check
```

### **Initialize Secret Enclave**

Initialize `/opt/secret/.sgx_secrets`:

```bash
mkdir -p /opt/secret/.sgx_secrets
```

You can choose between two methods, **3a (automatic) or 3b (manual)**:

#### **Initialize Secret Enclave - Automatic Registration (EXPERIMENTAL)**

{% hint style="danger" %}
WARNING: This method is experimental, and may not work. If it doesn't work, skip to step 3b.
{% endhint %}

The following commands will create the necessary environment variables and attempt to automatically register the node.

```bash
export SCRT_ENCLAVE_DIR=/usr/lib
export SCRT_SGX_STORAGE=/opt/secret/.sgx_secrets
secretd auto-register --pulsar-2
```

If this step was successful, you can skip straight to [step 9](run-a-full-node.md#9.-optimization).

#### **Initialize Secret Enclave - Manual Registration**

```bash
secretd init-enclave
```

### Verify Enclave Initialization

Attestation certificate should have been created by the previous step

```bash
ls -lh /opt/secret/.sgx_secrets/attestation_cert.der
```

Verify the certificate is valid. A 64 character registration key will be printed if it was successful.

```bash
PUBLIC_KEY=$(secretd parse /opt/secret/.sgx_secrets/attestation_cert.der  2> /dev/null | cut -c 3-)
echo $PUBLIC_KEY
```

### **Configure `secretd`**

{% hint style="info" %}
The following steps should use `secretd` be ran on the full node itself. To run the steps with `secretd` on a local machine, [set up the CLI](https://docs.scrt.network/cli/install-cli.html) there.
{% endhint %}

Configure `secretd`. Initially you'll be using the bootstrap node, as you'll need to connect to a running node and your own node is not running yet.

```bash
secretd config chain-id pulsar-2
secretd config node https://rpc.pulsar.scrttestnet.com
secretd config output json
```

### **Fund Secret Wallet**

If you already have a wallet funded with `SCRT`, you can import the wallet by doing the following:

```bash
secretd keys add <key-alias> --recover
```

Otherwise, you will need to set up a key. Make sure you back up the mnemonic and the keyring password.

```bash
secretd keys add <key-alias>
```

This will output your address, a 45 character-string starting with `secret1...`. Copy/paste it to get some test-SCRT from [the faucet](https://faucet.secrettestnet.io/). Continue when you have confirmed your account has some test-SCRT in it.

### **Configure Node Attestation**

1. Register your node on-chain

```bash
secretd tx register auth /opt/secret/.sgx_secrets/attestation_cert.der -y --from <key-alias>
```

2\. Pull & check your node's encrypted seed from the network

```bash
SEED=$(secretd query register seed $PUBLIC_KEY | cut -c 3-)
echo $SEED
```

3\. Get additional network parameters

These are necessary to configure the node before it starts.

```bash
secretd query register secret-network-params
ls -lh ./io-master-cert.der ./node-master-cert.der
```

### **Configure Your Secret Node**

{% hint style="info" %}
From here on, commands must be ran on the full node.
{% endhint %}

```bash
mkdir -p ~/.secretd/.node
secretd configure-secret node-master-cert.der $SEED
```

#### Add Seeds And Persistent Peers To Configuration File. <a href="#_16-add-persistent-peers-to-your-configuration-file" id="_16-add-persistent-peers-to-your-configuration-file"></a>

```bash
# seeds
perl -i -pe 's/seeds = ""/seeds = "7a421a6f5f1618f7b6fdfbe4854985746f85d263\@108.62.104.102:26656,a72e376dca664bac55e8ce55a2e972a8ae2c995e\@144.202.126.98:26656,a941999e72f4726d276ef055a09cb8bedf8e7a9a\@45.35.77.30:26656,f95ba3da4a9eec559397f4b47b1539e24af6904c\@52.190.249.47:26656"/' ~/.secretd/config/config.toml

# persistent_peers
perl -i -pe 's/persistent_peers = ""/persistent_peers = "7a421a6f5f1618f7b6fdfbe4854985746f85d263\@108.62.104.102:26656,a72e376dca664bac55e8ce55a2e972a8ae2c995e\@144.202.126.98:26656,a941999e72f4726d276ef055a09cb8bedf8e7a9a\@45.35.77.30:26656,f95ba3da4a9eec559397f4b47b1539e24af6904c\@52.190.249.47:26656"/' ~/.secretd/config/config.toml
```

### **Optimization**

In order to be able to handle NFT minting and other Secret Contract-heavy operations, it's recommended to update your SGX memory enclave cache:

```bash
sed -i.bak -e "s/^contract-memory-enclave-cache-size *=.*/contract-memory-enclave-cache-size = \"15\"/" ~/.secretd/config/app.toml
```

Also checkout[ this document](https://gist.github.com/blockpane/40bc6b64caa48fdaff3b0760acb51eaa) by `[ block pane ]` for fine tuning your machine for better uptime.

### **Set `minimum-gas-price` Parameter**

We recommend `0.0125uscrt` per gas unit:

```bash
perl -i -pe 's/^minimum-gas-prices = .+?$/minimum-gas-prices = "0.0125uscrt"/' ~/.secretd/config/app.toml
```

Your node will not accept transactions that specify `--fees` lower than the `minimun-gas-price` you set here.

### **Enable `secret-node`:**

{% hint style="info" %}
Note that the `secret-node` system file is created when installing sgx.
{% endhint %}

```bash
sudo systemctl enable secret-node && sudo systemctl start secret-node
```

If everything above worked correctly, the following command will show your node streaming blocks (this is for debugging purposes only, kill this command anytime with Ctrl-C).

```bash
journalctl -f -u secret-node
```

```bash
-- Logs begin at Mon 2020-02-10 16:41:59 UTC. --
Nov 09 11:16:31 scrt-node-01 secretd[619529]: 11:16AM INF indexed block height=12 module=txindex
Nov 09 11:16:35 scrt-node-01 secretd[619529]: 11:16AM INF Ensure peers module=pex numDialing=0 numInPeers=0 numOutPeers=0 numToDial=10
Nov 09 11:16:35 scrt-node-01 secretd[619529]: 11:16AM INF No addresses to dial. Falling back to seeds module=pex
Nov 09 11:16:36 scrt-node-01 secretd[619529]: 11:16AM INF Timed out dur=4983.86819 height=13 module=consensus round=0 step=1
Nov 09 11:16:36 scrt-node-01 secretd[619529]: 11:16AM INF received proposal module=consensus proposal={"Type":32,"block_id":{"hash":"0AF9693538AB0C753A7EA16CB618C5D988CD7DC01D63742DC4795606D10F0CA4","parts":{"hash":"58F6211ED5D6795E2AE4D3B9DBB1280AD92B2EE4EEBAA2910F707C104258D2A0","total":1}},"height":13,"pol_round":-1,"round":0,"signature":"eHY9dH8dG5hElNEGbw1U5rWqPp7nXC/VvOlAbF4DeUQu/+q7xv5nmc0ULljGEQR8G9fhHaMQuKjgrxP2KsGICg==","timestamp":"2021-11-09T11:16:36.7744083Z"}
Nov 09 11:16:36 scrt-node-01 secretd[619529]: 11:16AM INF received complete proposal block hash=0AF9693538AB0C753A7EA16CB618C5D988CD7DC01D63742DC4795606D10F0CA4 height=13 module=consensus
Nov 09 11:16:36 scrt-node-01 secretd[619529]: 11:16AM INF finalizing commit of block hash=0AF9693538AB0C753A7EA16CB618C5D988CD7DC01D63742DC4795606D10F0CA4 height=13 module=consensus num_txs=0 root=E4968C9B525DADA22A346D5E158C648BC561EEC351F402A611B9DA2706FD8267
Nov 09 11:16:36 scrt-node-01 secretd[619529]: 11:16AM INF minted coins from module account amount=6268801uscrt from=mint module=x/bank
Nov 09 11:16:36 scrt-node-01 secretd[619529]: 11:16AM INF executed block height=13 module=state num_invalid_txs=0 num_valid_txs=0
Nov 09 11:16:36 scrt-node-01 secretd[619529]: 11:16AM INF commit synced commit=436F6D6D697449447B5B373520353520323020352032342031312032333820353320383720313137203133372031323020313638203234302035302032323020353720343520363620313832203138392032333920393920323439203736203338203131322035342032332033203233362034375D3A447D
Nov 09 11:16:36 scrt-node-01 secretd[619529]: 11:16AM INF committed state app_hash=4B371405180BEE3557758978A8F032DC392D42B6BDEF63F94C2670361703EC2F height=13 module=state num_txs=0
^C
```

You are now a full node. 🎉

### Get Node ID <a href="#_21-get-your-node-id-with" id="_21-get-your-node-id-with"></a>

`secretd tendermint show-node-id`

And publish yourself as a node with this ID:

```bash
<your-node-id>@<your-public-ip>:26656
```

Be sure to point your CLI to your running node instead of the bootstrap node

`secretcli config node tcp://localhost:26657`

If someone wants to add you as a peer, have them add the above address to their persistent\_peers in their \~/.secretd/config/config.toml.

And if someone wants to use your node from their secretcli then have them run:

```bash
secretcli config chain-id pulsar-2
secretcli config output json
secretcli config indent true
secretcli config node tcp://<your-public-ip>:26657
```

### **State Sync**

You can skip syncing from scratch or download a snapshot by [State Syncing](https://docs.scrt.network/node-guides/state-sync.html#mainnet-state-sync) to the current block.

### **Optional: Become a Validator**

To turn your full node into a validator, see [Join Testnet as a Validator](run-a-full-node.md#how-to-join-secret-network-as-a-full-node-on-testnet).

***
