# Horcrux Dashboard Metrics Reference

Reference guide for all metrics exposed by Horcrux and displayed in the [Cumulo Horcrux Monitoring dashboard](https://github.com/Cumulo-pro/cumulo-horcrux/blob/main/grafana/Horcrux%20Monitoring%20%C2%B7%20Cumulo.Pro-1783181889803.json). Metrics are grouped by the dashboard sections in which they appear.

+info https://github.com/strangelove-ventures/horcrux

---

## Status metrics

These metrics indicate whether the signing cluster is operating correctly. In normal operation, error counters should always be **0**. Any value above 0 requires immediate investigation.

---

### Invalid combined signatures

`signer_error_total_invalid_signatures`

Total number of times the combined threshold signature produced by the cosigner cluster was invalid. The ideal value is always 0. Any increase indicates a signing failure that prevented a valid signature from being produced.

| Expected value | Action if non-zero |
|---------------|-------------------|
| 0 | Investigate cosigner logs immediately |

---

### Cosigners below threshold

`signer_error_total_insufficient_cosigners`

Total number of times the number of available cosigners did not reach the threshold required to produce a valid signature (2-of-3 in Cumulo's setup). This means signing could not proceed for that block.

| Expected value | Action if non-zero |
|---------------|-------------------|
| 0 | Check cosigner connectivity and Raft leader status |

---

## Signing metrics

These metrics reflect the active signing state of the cluster. They are the primary indicators of whether each chain is being signed correctly.

---

### Last block height signed

`signer_last_precommit_height`

The last block height for which a precommit was signed. This value should increase continuously with each new block. A stalled height indicates the cosigner cluster is not signing for that chain.

The dashboard shows `max by (chain)`, one value per chain, representing the most recent height signed across all cosigners.

| Expected behavior | Problem indicator |
|------------------|-------------------|
| Increases with each block | Stays static for more than a few block times |

---

### Last precommit round

`signer_last_precommit_round`

The consensus round in which the last precommit was signed. Rounds are iterations within the same block height when the network needs additional attempts to reach consensus.

A value of **0** means consensus was reached in the first round, which is ideal. Higher values are not necessarily a problem, they reflect network-level consensus behavior, not cosigner issues, but sustained high rounds may warrant investigation.

| Expected value | Note |
|---------------|------|
| 0 (most blocks) | Occasional higher values are normal |

---

### Seconds since last sign start

`signer_seconds_since_last_local_sign_start_time`

Time in seconds since the last local signing process was initiated on a cosigner. This value may increase beyond block time and is **rarely significant on its own**. It is included for completeness but should not trigger alerts in isolation.

---

### Seconds since last sign finish

`signer_seconds_since_last_local_sign_finish_time`

Time in seconds since the last local signing process completed on a cosigner. This value should remain below 2× the block time of each chain. A sustained high value indicates signing delays.

| Chain | Approximate block time | Alert threshold |
|-------|----------------------|----------------|
| cosmoshub-4 | ~6s | >12s |
| dymension_1100-1 | ~6s | >12s |
| mocha-4 | ~6s | >12s |
| seda-1 | ~6s | >12s |

---

### Seconds since last ephemeral share

`signer_seconds_since_last_local_ephemeral_share_time`

Each block, cosigners exchange ephemeral nonce secrets (one-time cryptographic values) before signing. This metric measures the time since the last exchange on a given cosigner.

This value **should not exceed the block time**. If it does, a cosigner missed a block cycle, this may indicate a Raft joining issue or connectivity problem between cosigners.

The dashboard shows `max by (chain)`, the worst-case value per chain across all cosigners.

| Expected behavior | Problem indicator |
|------------------|-------------------|
| Below block time (~6s) | Exceeds block time consistently |

---

### Time since last precommit

`signer_seconds_since_last_precommit`

Seconds elapsed since the last precommit was signed. Useful for single-signer setups or as a secondary indicator in threshold setups. Complements `signer_last_precommit_height` with a time dimension.

---

### Time since last prevote

`signer_seconds_since_last_prevote`

Seconds elapsed since the last prevote was signed. Useful for understanding the full CometBFT consensus cycle activity on a cosigner.

---

## Cosigner health metrics

These metrics reveal problems with cosigner communication and are the key indicators for diagnosing which specific cosigner is causing issues.

---

### Consecutive ephemeral shares missed

`signer_missed_ephemeral_shares`

Number of consecutive ephemeral shares missed from a specific peer cosigner, as seen by the Raft leader. Occasional misses are normal. A sustained increase indicates the leader cannot reach that cosigner, this points to connectivity, latency, or resource issues on the affected peer.

The dashboard shows one line per peer, identified by `peerid` (the cosigner's p2p address).

| Expected value | Action |
|---------------|--------|
| 0 or occasional spikes | Investigate connectivity if sustained above 3 |

---

### Total ephemeral shares lost

`signer_total_missed_ephemeral_shares`

Cumulative total of ephemeral signature parts lost since the cosigner started. Unlike `signer_missed_ephemeral_shares` (consecutive), this is a running total. A significant count may indicate recurring connectivity problems.

---

### Total nonces requested when cache is drained

`signer_total_drained_nonce_cache`

Total times a cosigner requested nonces when its local cache was exhausted. May indicate high signing volume or insufficient nonce pre-generation. Rarely critical but useful for diagnosing performance under load.

---

## Signing performance metrics

These metrics quantify how quickly the cluster produces signatures. They are most useful when diagnosing latency issues. Available only on the **Raft leader**, followers report `NaN`.

---

### Time to reach signing threshold

`signer_sign_block_threshold_lag_seconds`

Time in seconds for the cluster to collect signatures from the minimum required cosigners (2-of-3). The dashboard uses the **p90 quantile**, 90% of blocks should be signed within this time. Lower values indicate a more responsive cluster.

---

### Time to collect all cosigner signatures

`signer_sign_block_cosigner_lag_seconds`

Time in seconds to collect signatures from **all** cosigners after the threshold has been reached. Should be very low, high values indicate latency between the leader and a specific cosigner.

---

### Per-cosigner signature lag

`signer_cosigner_sign_lag_seconds`

Time taken to receive the signature from each individual cosigner, as observed by the Raft leader (p90 quantile). Useful for pinpointing which specific cosigner is slow when `signer_sign_block_cosigner_lag_seconds` is elevated. Reports `NaN` on followers.

---

## Raft cluster metrics

These metrics reflect the health and stability of the Raft consensus layer that coordinates the cosigner cluster.

---

### Times elected Raft leader

`signer_total_raft_leader`

Total number of times each cosigner has been elected as the Raft leader. Should be roughly balanced across all three cosigners over time. A large imbalance may indicate one cosigner is always preferred or that others are frequently unavailable.

---

### Times acting as Raft follower

`signer_total_raft_not_leader`

Total number of times a cosigner acted as a Raft follower, forwarding its signature to the leader rather than coordinating signing itself. Complements `signer_total_raft_leader`, the two values across all cosigners should add up consistently.

---

### Raft leader election timeouts

`signer_total_raft_leader_election_timeout`

Total number of times a Raft leader election failed because not enough peers responded within the timeout. Should always be **0**. Any increase indicates cluster instability, likely connectivity issues between cosigners or a cosigner being unreachable.

| Expected value | Action if non-zero |
|---------------|-------------------|
| 0 | Check inter-cosigner connectivity on port 2222 |

---

### Raft apply

`horcrux_raft_apply`

Counter for the number of log entries successfully applied via Raft consensus. A continuously increasing value indicates the Raft cluster is healthy and processing entries. If this stalls, the cluster may have lost quorum.

---

## Sentry connectivity metrics

These metrics track TCP connections between cosigners and their sentry/validator nodes. Spikes indicate reconnection attempts.

---

### Consecutive sentry connection retries

`signer_sentry_connect_tries`

Number of consecutive TCP connection attempts from a cosigner to a specific sentry node. Resets to 0 after a successful connection. A sustained high value indicates the cosigner cannot reach that sentry, check firewall rules and sentry availability.

The dashboard shows per cosigner per sentry: `{{instance}} → {{node}}`.

---

### Total sentry connection attempts

`signer_total_sentry_connect_tries`

Cumulative total of TCP connection attempts to sentry nodes since the cosigner started. Unlike `signer_sentry_connect_tries`, this does not reset after a successful connection. A high total may indicate the cosigner or sentry has restarted frequently, or that connection stability has been poor.

---

## Related guides

- [Horcrux metrics with Grafana & Prometheus](./metrics-grafana-prometheus.md)
- [Useful Commands](./useful-commands.md)
- [Installation and Usage Guide](./installation-guide.md)
