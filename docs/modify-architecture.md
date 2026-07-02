# Modify an existing Horcrux architecture

This guide covers the migration from a **single-signer architecture** (1 validator node + 2 sentries) to a **2-of-3 threshold signing architecture** (3 cosigner nodes + 3 sentries). This was Cumulo's migration path when adopting Horcrux on cosmoshub-4.

If you are setting up Horcrux from scratch, see the [Installation and Usage Guide](./installation-guide.md) instead.

+info https://github.com/strangelove-ventures/horcrux

---

## Before and after

| | Before | After |
|-|--------|-------|
| Signing | Single validator node | 3 cosigner nodes (2-of-3 threshold) |
| Sentries | 2 | 3 |
| Key storage | `priv_validator_key.json` on validator | Ed25519 shards distributed across cosigners |
| Single point of failure | Yes | No |

---

## Prerequisites

- Existing validator node running and in sync
- 3 new VPS/servers provisioned for cosigner nodes
- 1 additional sentry node (to reach 3 total)
- Horcrux installed on all 3 cosigner nodes — see [Installation and Usage Guide](./installation-guide.md)

---

## 1️⃣ Back up the existing setup

Before making any changes, back up the current validator home directory and key files.

```bash
# On the existing validator node
cp -r .<NODE_HOME> .<NODE_HOME>_bak
sha256sum .<NODE_HOME>/config/priv_validator_key.json
sha256sum .<NODE_HOME>/data/priv_validator_state.json
```

> ❗ Keep the checksum output. Verify it against the backup copy before proceeding.

---

## 2️⃣ Initialize Horcrux configuration on each cosigner

Run on **each of the 3 cosigner nodes**. This generates `config.yaml` and the ECIES communication keys for that node.

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

> ⚠️ Usernames may differ per cosigner node depending on provider and server configuration. Always confirm the actual username on each node before running remote commands.

Verify the generated config:

```bash
cat ~/.horcrux/config.yaml
```

<!-- imagen: ejemplo de config.yaml generado en cosigner -->

---

## 3️⃣ Generate ECIES keys

ECIES keys encrypt communication between cosigners. They are generated automatically by `horcrux config init` in step 2. Verify they exist on each cosigner:

```bash
ls ~/.horcrux/
# Expected: config.yaml  ecies_keys.json
```

> ⚠️ **Never copy `ecies_keys.json` between cosigners.** Each node must have its own unique ECIES key pair. If any node is missing this file, re-run `horcrux config init` on that node.

---

## 4️⃣ Generate and distribute Ed25519 shards

Run on a **secure machine** with access to `priv_validator_key.json`. This is the same key currently on your validator node.

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

### Verify shard IDs before distributing

```bash
cat cosigner_1/<chain-id>_shard.json | \
  python3 -c "import json, sys; d = json.load(sys.stdin); print('shardID:', d['id'])"
```

Expected: `shardID: 1` for cosigner_1, `shardID: 2` for cosigner_2, `shardID: 3` for cosigner_3.

### Distribute shards to each cosigner

```bash
scp cosigner_1/<chain-id>_shard.json <user>@<signer-ip-1>:~/.horcrux/
scp cosigner_2/<chain-id>_shard.json <user>@<signer-ip-2>:~/.horcrux/
scp cosigner_3/<chain-id>_shard.json <user>@<signer-ip-3>:~/.horcrux/
```

Verify on each cosigner:

```bash
ls ~/.horcrux/
# Expected: config.yaml  ecies_keys.json  <chain-id>_shard.json
```

---

## 5️⃣ Share consensus state between cosigners

Stop the existing validator node and copy its current state to each cosigner. This prevents signing at an already-signed height.

```bash
# On the existing validator node
sudo systemctl stop <chain-noded>
cat .<NODE_HOME>/data/priv_validator_state.json
```

Note the `height` value. On **each cosigner node**:

```bash
mkdir -p ~/.horcrux/state
echo '{"height":"<HEIGHT>","round":"0","step":3}' \
  > ~/.horcrux/state/<chain-id>_priv_validator_state.json

# Verify
cat ~/.horcrux/state/<chain-id>_priv_validator_state.json
```

> ⚠️ Use the height from the validator node. The `"round"` value must be a string.

---

## 6️⃣ Back up and clean the existing Horcrux directory (if any)

If the validator node previously had a `.horcrux` directory from an earlier setup:

```bash
# On the existing validator node
cp -r ~/.horcrux ~/.horcrux_bak
sha256sum ~/.horcrux_bak/ecies_keys.json   # verify backup is intact
rm -rf ~/.horcrux
```

---

## 7️⃣ Configure all sentry nodes

On **each of the 3 sentry nodes**, enable the private validator listener:

```bash
sed -i 's#priv_validator_laddr = ""#priv_validator_laddr = "tcp://0.0.0.0:1234"#g' \
  .<NODE_HOME>/config/config.toml

# Verify
grep priv_validator_laddr .<NODE_HOME>/config/config.toml
```

Expected output:

```
priv_validator_laddr = "tcp://0.0.0.0:1234"
```

> ❗ **REMOVE `priv_validator_key.json` from all sentry nodes.** The key is now managed exclusively by the Horcrux cosigner cluster.

```bash
rm .<NODE_HOME>/config/priv_validator_key.json
```

---

## 8️⃣ Start Horcrux and restart sentry nodes

Start Horcrux on **all three cosigners** at approximately the same time:

```bash
sudo systemctl start horcrux
sudo journalctl -u horcrux -f
```

Then restart all sentry nodes:

```bash
sudo systemctl restart <chain-noded>
sudo journalctl -u <chain-noded> -f
```

<!-- imagen: logs de Horcrux mostrando sign requests y conexiones a los tres sentries -->

---

## ✅ Verification

```bash
# Horcrux logs — look for sign confirmations across all cosigners
sudo journalctl -u horcrux -f | grep -E "(sign|error|connected)"

# Validator state — height should increase with each block
cat ~/.horcrux/state/<chain-id>_priv_validator_state.json
```

---

## File layout after migration

Each cosigner node:

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

- [Installation and Usage Guide](./installation-guide.md)
- [Adding a chain to an existing multi-chain setup](./add-chain-multichain.md)
- [Horcrux Useful Commands](./useful-commands.md)
- [Signer Migration Runbook](./signer-migration-runbook.md)
