# Adding a chain to an existing multi-chain Horcrux setup

This guide explains how to integrate a new chain into a Horcrux cosigner cluster that is already running and signing for one or more chains. The process does not require regenerating ECIES keys or reinitializing the full configuration — only the Ed25519 shards for the new chain need to be created.

+info https://github.com/strangelove-ventures/horcrux

---

## Context

In a multi-chain Horcrux setup, all cosigners share a single `config.yaml` and a single `horcrux.service`. Each chain is represented by:

- A **shard file**: `~/.horcrux/<CHAIN_ID>_shard.json`
- A **state file**: `~/.horcrux/state/<CHAIN_ID>_priv_validator_state.json`
- A **node entry** in `config.yaml` pointing to the chain's sentry/validator

Adding a new chain means distributing new shard files, updating `config.yaml` on all cosigners, and restarting Horcrux.

> ⚠️ **Important:** Usernames and paths may differ per cosigner node depending on provider and server configuration. Always confirm the actual username on each node before running remote commands — do not assume a uniform `ubuntu` or `$USER`.

---

## Node reference

| Node | User | Shard ID |
|------|------|----------|
| signer1 | `<user-signer1>` | 1 |
| signer2 | `<user-signer2>` | 2 |
| signer3 | `<user-signer3>` | 3 |

---

## 1️⃣ Generate Ed25519 shards for the new chain

Run this on the machine that has secure access to the new chain's `priv_validator_key.json`. Copy the key into your working directory first.

❗️ CAUTION: this is a delicate operation. The source machine has momentary access to the full private key. Perform it in a secure environment and move the key file to a safe location immediately after.

```
horcrux create-ed25519-shards \
  --chain-id <new-chain-id> \
  --key-file ./priv_validator_key.json \
  --threshold 2 \
  --shards 3
```

Files generated:

```
ls -R
```

*./cosigner_1:*
*<NEW_CHAIN_ID>_shard.json*
*./cosigner_2:*
*<NEW_CHAIN_ID>_shard.json*
*./cosigner_3:*
*<NEW_CHAIN_ID>_shard.json*

✔️ Move `priv_validator_key.json` to a safe location — it is no longer needed on this machine.

---

## 2️⃣ Verify shard IDs before distributing

Confirm that each shard file has the expected `id` before copying to the cosigner nodes. If `jq` is not available, use Python:

```
cat cosigner_1/<NEW_CHAIN_ID>_shard.json | \
  python3 -c "import json, sys; d = json.load(sys.stdin); print('shardID:', d['id'])"
```

Expected output for `cosigner_1`: `shardID: 1`, for `cosigner_2`: `shardID: 2`, etc.

---

## 3️⃣ Distribute shards to each cosigner node

Copy each `cosigner_N/<CHAIN_ID>_shard.json` into `~/.horcrux/` on the corresponding node. The existing `ecies_keys.json` is reused — do not replace it.

```
scp cosigner_1/<CHAIN_ID>_shard.json <user-signer1>@<ip-signer1>:~/.horcrux/
scp cosigner_2/<CHAIN_ID>_shard.json <user-signer2>@<ip-signer2>:~/.horcrux/
scp cosigner_3/<CHAIN_ID>_shard.json <user-signer3>@<ip-signer3>:~/.horcrux/
```

Verify on each node that the file is in place:

```
ls ~/.horcrux/
```

---

## 4️⃣ Initialize the state file on each cosigner

Each cosigner needs a `priv_validator_state.json` for the new chain. First, get the current height from the validator node:

```
sudo systemctl stop <new-chain-noded>
cat .<NODE_HOME>/data/priv_validator_state.json
```

Example output:
```
{
  "height": "5123400",
  "round": 0,
  "step": 3
}
```

✔️ On **each** cosigner node, create the state file using that height. Note the `"round"` value must be a string:

```
echo '{"height":"5123400","round":"0","step":3}' \
  > ~/.horcrux/state/<NEW_CHAIN_ID>_priv_validator_state.json
```

Verify:

```
cat ~/.horcrux/state/<NEW_CHAIN_ID>_priv_validator_state.json
```

---

## 5️⃣ Update config.yaml on all cosigner nodes

Add the new chain's entry to the `chains` section of `~/.horcrux/config.yaml`. This file must be **identical** on all three cosigners.

Example of an existing multi-chain `config.yaml` with the new chain added:

```yaml
cosigner:
  threshold: 2
  shards: 3
  p2pListen: "tcp://0.0.0.0:2222"
  peers:
    - remoteAddress: "tcp://<ip-signer1>:2222"
      shardID: 1
    - remoteAddress: "tcp://<ip-signer2>:2222"
      shardID: 2
    - remoteAddress: "tcp://<ip-signer3>:2222"
      shardID: 3
  grpcTimeout: "1000ms"
  raftTimeout: "1000ms"
chains:
  - chainID: "cosmoshub-4"
    key: "/home/<user>/.horcrux/cosmoshub-4_shard.json"
    state: "/home/<user>/.horcrux/state/cosmoshub-4_priv_validator_state.json"
    nodes:
      - address: "tcp://<sentry-ip>:1234"
  - chainID: "<new-chain-id>"                        # ← new entry
    key: "/home/<user>/.horcrux/<new-chain-id>_shard.json"
    state: "/home/<user>/.horcrux/state/<new-chain-id>_priv_validator_state.json"
    nodes:
      - address: "tcp://<new-chain-sentry-ip>:1234"
```

❗️ FOR ALL COSIGNER NODES, MAKE SURE THE `config.yaml` IS IDENTICAL AND THE SHARD FILES ARE IN PLACE BEFORE RESTARTING.

---

## 6️⃣ Configure the new chain's sentry node

On the new chain's sentry/validator node, enable the private validator listener:

```
sed -i 's#priv_validator_laddr = ""#priv_validator_laddr = "tcp://0.0.0.0:1234"#g' \
  .<NODE_HOME>/config/config.toml

cat .<NODE_HOME>/config/config.toml | grep priv_validator_laddr
```

*priv_validator_laddr = "tcp://0.0.0.0:1234"*

❗️ REMOVE THE ORIGINAL `priv_validator_key.json` FROM THE VALIDATOR NODE. The key is now managed exclusively by the Horcrux cluster.

---

## 7️⃣ Restart Horcrux and the new chain's node

Restart Horcrux on all three cosigners at approximately the same time:

```
sudo systemctl restart horcrux
sudo journalctl -u horcrux -f
```

Then start the new chain's validator node:

```
sudo systemctl restart <new-chain-noded>
sudo journalctl -u <new-chain-noded> -f
```

✔️ In the Horcrux logs, you should see successful connections to both the existing chains and the new one. Look for sign request confirmations:

```
sudo journalctl -u horcrux -f | grep -E "(sign|<new-chain-id>|error)"
```

---

## File layout after adding the new chain

Each cosigner node should have the following structure:

```
~/.horcrux/
├── config.yaml
├── ecies_keys.json
├── cosmoshub-4_shard.json
├── <new-chain-id>_shard.json
└── state/
    ├── cosmoshub-4_priv_validator_state.json
    └── <new-chain-id>_priv_validator_state.json
```
