# Horcrux metrics with Grafana & Prometheus

This guide explains how to configure Prometheus to scrape Horcrux metrics from your cosigner nodes and import the Cumulo dashboard into Grafana.

+info https://github.com/strangelove-ventures/horcrux

---

## 1️⃣ Enable the metrics port on each cosigner node

On **each cosigner node**, edit the Horcrux config to expose the Prometheus endpoint:

```bash
sudo vi ~/.horcrux/config.yaml
```

Add or update the `debugAddr` field:

```yaml
debugAddr: "0.0.0.0:6001"
```

---

## 2️⃣ Open the firewall port on each cosigner

Allow the Prometheus server to reach port `6001` on each cosigner. Replace `<prometheus-ip>` with the IP of your Prometheus server:

```bash
sudo ufw allow from <prometheus-ip> to any port 6001
```

> ⚠️ Do not open port 6001 publicly. Restrict access to the Prometheus server IP only.

Verify the port is listening after restarting Horcrux:

```bash
ss -tlnp | grep 6001
curl -s http://localhost:6001/metrics | head -10
```

---

## 3️⃣ Configure Prometheus scrape jobs

On the Prometheus server, edit the configuration file:

```bash
sudo vi /etc/prometheus/prometheus.yml
```

Add one job per Horcrux cluster. Each target entry must include `chain` and `share` labels: these are used by the dashboard to filter and color metrics per chain and cosigner.

The same cosigner IPs can appear multiple times if they sign multiple chains:

```yaml
global:
  scrape_interval: 3s   # Lower interval for more granular signing metrics

scrape_configs:
  - job_name: horcrux_cosmoshub
    static_configs:
      - targets: ["<signer-ip-1>:6001"]
        labels: { "chain":"cosmoshub-4", "share":"1" }
      - targets: ["<signer-ip-2>:6001"]
        labels: { "chain":"cosmoshub-4", "share":"2" }
      - targets: ["<signer-ip-3>:6001"]
        labels: { "chain":"cosmoshub-4", "share":"3" }

  - job_name: horcrux_celestia
    static_configs:
      - targets: ["<signer-ip-1>:6001"]
        labels: { "chain":"celestia", "share":"1" }
      - targets: ["<signer-ip-2>:6001"]
        labels: { "chain":"celestia", "share":"2" }
      - targets: ["<signer-ip-3>:6001"]
        labels: { "chain":"celestia", "share":"3" }
```

> ℹ️ For multi-chain setups where the same 3 cosigners sign multiple chains, repeat the target block for each chain with the corresponding `chain` label. The `job` label is used in the dashboard to select which cluster to monitor.

Restart Prometheus:

```bash
sudo systemctl restart prometheus
sudo journalctl -u prometheus -f --no-hostname -o cat
```

Verify targets are UP:

```bash
curl -s http://localhost:<prometheus-port>/api/v1/targets | \
  python3 -c "
import json,sys
d=json.load(sys.stdin)
for t in d['data']['activeTargets']:
    print(t['labels'].get('job','?'), t['labels'].get('instance','?'), t['health'])
"
```

---

## 4️⃣ Restart Horcrux on each cosigner

```bash
sudo systemctl restart horcrux
sudo journalctl -u horcrux -f
```

---

## 5️⃣ Verify metrics are available

From the Prometheus server, check that metrics are being scraped:

```bash
curl -s http://<signer-ip>:6001/metrics | grep signer_last_precommit_height
```

You should see output like:

```
signer_last_precommit_height{chain_id="cosmoshub-4"} 2.1345678e+07
```

---

## 6️⃣ Import the Cumulo dashboard in Grafana

Download the dashboard JSON from the repo:

```
https://github.com/Cumulo-pro/cumulo-horcrux/blob/main/grafana/cumulo-horcrux-dashboard.json
```

In Grafana: **Dashboards → Import → Upload JSON file**

When prompted, assign your Prometheus datasource. The dashboard uses the standard `__inputs` mechanism: Grafana will ask you to map `DS_PROMETHEUS` to your datasource on import.

<!-- imagen: pantalla de import de Grafana mostrando el selector de datasource -->

---

## 7️⃣ Using the dashboard

Once imported, the dashboard has two variables in the top bar:

- **Datasource**: your Prometheus instance
- **Job**: select which Horcrux cluster to monitor (e.g. `horcrux_cosmoshub`, `horcrux_celestia`)

The dashboard is organized in three sections:

**Metrics that don't always correspond to block time**: signing timestamps and last signed height per chain. These metrics may fluctuate beyond block time without indicating a problem.

**Watching For Cosigner Trouble**: precommit rounds and missed ephemeral shares. These are the key indicators of cosigner connectivity issues.

**Checking Signing Performance**: threshold lag and cosigner lag metrics. Expand this section when something looks wrong in the sections above.

<!-- imagen: captura del dashboard Cumulo con datos reales -->

---

## Troubleshooting

**Job variable is empty in the dashboard**

The variable queries `label_values(signer_last_precommit_height, job)`: if it returns nothing, Prometheus is not receiving metrics from the cosigners. Check:

```bash
# From Prometheus server
curl -s http://<signer-ip>:6001/metrics | head -5
```

If the curl times out or fails, the firewall on the cosigner is blocking access. Open the port from the Prometheus server IP.

**Metrics show only one chain despite multiple chains configured**

Each chain needs its own target entry in `prometheus.yml` with the correct `chain` label. The same cosigner IP can appear multiple times: once per chain it signs.

**`debugAddr` not found in config.yaml**

Add it manually at the root level of the config:

```yaml
debugAddr: "0.0.0.0:6001"
```

---

## Related guides

- [Installation and Usage Guide](./installation-guide.md)
- [Useful Commands](./useful-commands.md)
- [Dashboard metrics reference](./metrics-dashboard.md)
