# cumulo-horcrux

Horcrux threshold signing architecture, deployment guides, and Grafana metrics dashboard for Cumulo's CometBFT validators.

<!-- imagen: arquitectura general Cumulo Horcrux -->

## What is Horcrux?

Horcrux is a multi-party-computation (MPC) signing service for CometBFT nodes. It enhances security and availability by distributing the validator private key among multiple cosigner nodes using Shamir's Secret Sharing, thereby preventing double signing and eliminating single points of failure. The project uses Raft for leader election and consensus among cosigners.

Cumulo runs a **2-of-3 threshold signing** setup across three independent cosigner nodes on separate providers, signing multiple chains from a single Horcrux cluster.

+info https://github.com/strangelove-ventures/horcrux

---

## Documentation

[![Horcrux Installation and Usage Guide](https://img.shields.io/badge/-Horcrux%20Installation%20and%20Usage%20Guide-FFA500?style=for-the-badge&labelColor=321A0C&logoColor=white)](docs/installation-guide.md)

[![Modify an existing Horcrux architecture](https://img.shields.io/badge/-Modify%20an%20existing%20Horcrux%20architecture-FFA500?style=for-the-badge&labelColor=321A0C&logoColor=white)](docs/modify-architecture.md)

[![Adding a chain to an existing multi-chain setup](https://img.shields.io/badge/-Adding%20a%20chain%20to%20an%20existing%20multi--chain%20setup-FFA500?style=for-the-badge&labelColor=321A0C&logoColor=white)](docs/add-chain-multichain.md)

[![Horcrux Useful Commands](https://img.shields.io/badge/-Horcrux%20Useful%20Commands-A86A24?style=for-the-badge&labelColor=321A0C&logoColor=white)](docs/useful-commands.md)

[![Horcrux Metrics with Grafana & Prometheus](https://img.shields.io/badge/-Horcrux%20Metrics%20with%20Grafana%20%26%20Prometheus-321A0C?style=for-the-badge&labelColor=060502&logoColor=FFA500)](docs/metrics-grafana-prometheus.md)

[![Horcrux Dashboard Metrics Reference](https://img.shields.io/badge/-Horcrux%20Dashboard%20Metrics%20Reference-321A0C?style=for-the-badge&labelColor=060502&logoColor=FFA500)](docs/metrics-dashboard.md)

---

## Repository structure

```
cumulo-horcrux/
├── README.md
├── docs/
│   ├── installation-guide.md
│   ├── modify-architecture.md
│   ├── add-chain-multichain.md
│   ├── useful-commands.md
│   ├── metrics-grafana-prometheus.md
│   └── metrics-dashboard.md
└── grafana/
    └── cumulo-horcrux-dashboard.json
```

---

## Grafana dashboard

The Cumulo Horcrux Monitoring dashboard is available in [`grafana/cumulo-horcrux-dashboard.json`](grafana/cumulo-horcrux-dashboard.json).

It uses a job-based variable to select which cosigner cluster to monitor, and displays metrics per chain automatically — no changes needed when adding or removing chains. See [Horcrux Metrics with Grafana & Prometheus](docs/metrics-grafana-prometheus.md) for setup instructions.

<!-- imagen: captura del dashboard Cumulo Horcrux Monitoring -->

---

## Cumulo.Pro

[cumulo.pro](https://cumulo.pro) · Stake with us.
