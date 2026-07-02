# Horcrux Useful Commands

Quick reference for operating and diagnosing a running Horcrux cosigner cluster.

---

## Service management

```bash
# Stop the service
sudo systemctl stop horcrux

# Start the service
sudo systemctl start horcrux

# Restart the service
sudo systemctl restart horcrux

# Check service status
sudo systemctl status horcrux
```

---

## Logs

```bash
# Follow live logs
sudo journalctl -u horcrux -f

# Filter for signing activity only
sudo journalctl -u horcrux -f | grep -E "(sign|error|connected)"

# Filter by chain ID in a multi-chain setup
sudo journalctl -u horcrux -f | grep "<chain-id>"

# Last 100 lines
sudo journalctl -u horcrux -n 100 --no-pager
```

---

## Version

```bash
horcrux version
```

<!-- imagen: output de horcrux version -->

---

## Raft leader

```bash
# Show the current Raft leader
horcrux leader
```

<!-- imagen: output de horcrux leader -->

```bash
# Force election of a specific cosigner as leader (by shard ID)
horcrux elect 2
```

<!-- imagen: output de horcrux elect -->

---

## State inspection

```bash
# List all state files (one per chain)
ls ~/.horcrux/state/

# View the current signing state for a specific chain
cat ~/.horcrux/state/<chain-id>_priv_validator_state.json

# Verify the height is advancing with each block
watch -n 5 "cat ~/.horcrux/state/<chain-id>_priv_validator_state.json"
```

---

## Multi-chain diagnostics

```bash
# Verify which chains are configured
cat ~/.horcrux/config.yaml | grep chainID

# Check that shard files exist for all configured chains
ls ~/.horcrux/ | grep shard

# Verify shard ID on a specific file
cat ~/.horcrux/<chain-id>_shard.json | \
  python3 -c "import json, sys; d = json.load(sys.stdin); print('shardID:', d['id'])"

# Check state files for all chains at once
for f in ~/.horcrux/state/*_priv_validator_state.json; do
  echo "--- $f ---"
  cat "$f"
done
```

---

## Configuration

```bash
# View full config
cat ~/.horcrux/config.yaml

# Verify ECIES keys exist
ls ~/.horcrux/ecies_keys.json

# Check open ports (cosigner p2p :2222, sentry listener :1234)
ss -tlnp | grep -E "(2222|1234)"
```

---

## Related guides

- [Installation and Usage Guide](./installation-guide.md)
- [Signer Migration Runbook](./signer-migration-runbook.md)
- [Adding a chain to an existing multi-chain setup](./add-chain-multichain.md)
- [Metrics with Grafana & Prometheus](./metrics-grafana-prometheus.md)
