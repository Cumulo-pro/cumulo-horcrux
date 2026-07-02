# Horcrux Installation and Usage Guide

Horcrux is a multi-party-computation (MPC) signing service for CometBFT nodes. It enhances security and availability by distributing private keys among multiple nodes, thereby preventing double signing and improving performance. The project also introduces Raft for leader election and consensus.

+info https://github.com/strangelove-ventures/horcrux

---

## Architecture overview

The reference architecture for Cumulo's Horcrux setup uses a **2-of-3 threshold signing** model:

- **3 sentry nodes** : public-facing, connected to the P2P network
- **3 cosigner nodes** : private, running the Horcrux process with one shard each

Each sentry connects to the cosigner cluster via a private network on port `:1234`. The cosigners communicate with each other on port `:2222` using ECIES-encrypted channels.

<!-- imagen: arquitectura general sentry → cosigner cluster -->

---

## Requirements

| Component | Minimum |
|-----------|---------|
| OS | Ubuntu 20.04+ |
| Go | 1.21+ |
| Horcrux | See [latest release](https://github.com/strangelove-ventures/horcrux/releases) |
| Nodes | 3 sentry nodes + 3 cosigner nodes |

---

## 1️⃣ Install Horcrux on each cosigner node

Check the latest release at https://github.com/strangelove-ventures/horcrux/releases and replace `<version>` accordingly.

```bash
wget https://github.com/strangelove-ventures/horcrux/releases/download/<version>/horcrux_linux-amd64
mv horcrux_linux-amd64 horcrux
sudo cp horcrux /usr/local/bin/
sudo chmod +x /usr/local/bin/horcrux
horcrux version
```

---

## 2️⃣ Initialize the cosigner configuration

Run on **each cosigner node**. This command generates the base `config.yaml` and the ECIES communication keys for that node.

```bash
horcrux config init \
  --node "tcp://<sentry-ip-1>:1234" \
  --node "tcp://<sentry-ip-2>:1234" \
  --node "tcp://<sentry-ip-3>:1234" \
  --cosigner "tcp://<signer-ip-1>:2222|1" \
  --cosigner "tcp://<signer-ip-2>:2222|2" \
  --cosigner "tcp://<signer-ip-3>:2222|3" \
  --threshold 2 \
  --grpc-timeout 1000ms \
  --raft-timeout 1000ms
```

> ⚠️ **Multi-chain setup:** `horcrux config init` generates a configuration for a **single chain**. If you intend to run Horcrux across multiple chains, use this command to generate the initial structure and ECIES keys, then edit `~/.horcrux/config.yaml` manually to add the `chains` array with all required chain entries. See [Adding a chain to an existing multi-chain setup](./add-chain-multichain.md) for details.

Verify the generated config:

```bash
cat ~/.horcrux/config.yaml
```

<!-- imagen: ejemplo de config.yaml generado -->

---

## 3️⃣ Generate and distribute Ed25519 shards

Run on a **secure machine** that has access to the chain's `priv_validator_key.json`. This machine has momentary access to the full private key - perform this in a secure environment.

```bash
horcrux create-ed25519-shards \
  --chain-id <chain-id> \
  --key-file ./priv_validator_key.json \
  --threshold 2 \
  --shards 3
```

Files generated:

```
./cosigner_1/<chain-id>_shard.json
./cosigner_2/<chain-id>_shard.json
./cosigner_3/<chain-id>_shard.json
```

### 3a️⃣ Verify shard IDs before distributing

Confirm that each shard file has the expected `id` before copying to the cosigner nodes:

```bash
cat cosigner_1/<chain-id>_shard.json | \
  python3 -c "import json, sys; d = json.load(sys.stdin); print('shardID:', d['id'])"
```

Expected output for `cosigner_1`: `shardID: 1`, for `cosigner_2`: `shardID: 2`, etc.

### 3b️⃣ Distribute shards to each cosigner

```bash
scp cosigner_1/<chain-id>_shard.json <user>@<signer-ip-1>:~/.horcrux/
scp cosigner_2/<chain-id>_shard.json <user>@<signer-ip-2>:~/.horcrux/
scp cosigner_3/<chain-id>_shard.json <user>@<signer-ip-3>:~/.horcrux/
```

> ⚠️ Usernames may differ per cosigner node depending on provider and server configuration. Always confirm the actual username on each node.

Verify on each cosigner that the file is in place:

```bash
ls ~/.horcrux/
```

✔️ Move `priv_validator_key.json` to a safe location - it is no longer needed on the generation machine.

---

## 4️⃣ Share consensus state between cosigners

Before starting Horcrux, copy the current `priv_validator_state.json` from the validator node to each cosigner. This prevents the node from signing at a height already signed.

On the **validator node**:

```bash
sudo systemctl stop <chain-noded>
cat .<NODE_HOME>/data/priv_validator_state.json
```

On **each cosigner node**, create the state file:

```bash
mkdir -p ~/.horcrux/state
echo '{"height":"<HEIGHT>","round":"0","step":3}' \
  > ~/.horcrux/state/<chain-id>_priv_validator_state.json
```

> ⚠️ Use the height from the validator node. The `"round"` value must be a string.

---

## 5️⃣ Create the Horcrux systemd service

On **each cosigner node**:

```bash
sudo tee /etc/systemd/system/horcrux.service > /dev/null << 'SERVICE'
[Unit]
Description=Horcrux Signing Service
After=network.target

[Service]
User=<user>
ExecStart=/usr/local/bin/horcrux start
Restart=always
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
SERVICE

sudo systemctl daemon-reload
sudo systemctl enable horcrux
```

---

## 6️⃣ Configure sentry nodes

On **each sentry node**, enable the private validator listener in `config.toml`:

```bash
sed -i 's#priv_validator_laddr = ""#priv_validator_laddr = "tcp://0.0.0.0:1234"#g' \
  .<NODE_HOME>/config/config.toml

# Verify
cat .<NODE_HOME>/config/config.toml | grep priv_validator_laddr
```

Expected output:

```
priv_validator_laddr = "tcp://0.0.0.0:1234"
```

> ❗ **REMOVE the `priv_validator_key.json` from each sentry node.** The key is now managed exclusively by the Horcrux cosigner cluster. Leaving it in place risks double signing.

---

## 7️⃣ Start Horcrux and the validator node

Start Horcrux on **all three cosigners** at approximately the same time:

```bash
sudo systemctl start horcrux
sudo journalctl -u horcrux -f
```

Then start the sentry/validator node:

```bash
sudo systemctl start <chain-noded>
sudo journalctl -u <chain-noded> -f
```

<!-- imagen: logs de Horcrux mostrando conexiones exitosas y sign requests -->

---

## ✅ Verification

Check that Horcrux is signing correctly:

```bash
# Horcrux logs - look for sign confirmations
sudo journalctl -u horcrux -f | grep -E "(sign|error|connected)"

# Validator state on each cosigner
cat ~/.horcrux/state/<chain-id>_priv_validator_state.json
```

The height in the state file should be increasing with each new block.

---

## File layout

Each cosigner node should have the following structure after setup:

```
~/.horcrux/
├── config.yaml
├── ecies_keys.json
├── <chain-id>_shard.json
└── state/
    └── <chain-id>_priv_validator_state.json
```

---

## Related guides

- [Modify an existing Horcrux architecture](./modify-architecture.md)
- [Adding a chain to an existing multi-chain setup](./add-chain-multichain.md)
- [Horcrux Useful Commands](./useful-commands.md)
- [Metrics with Grafana & Prometheus](./metrics-grafana-prometheus.md)
