# Self-Hosted Monitoring Stack with Docker Compose

A complete, production-ready monitoring setup based on Docker Compose. This stack collects, stores, and visualizes system metrics (CPU, memory, disk, network) across multiple Linux hosts with automatic HTTPS provided by Caddy.

---

## Architecture Overview

```text
[ Internet / Local Network ]
            │
            ▼  (Port 80 / 443 with TLS)
      ┌───────────┐
      │   Caddy   │ (Reverse Proxy & Automatic HTTPS via DuckDNS)
      └─────┬─────┘
            │  (Shared Docker Network: caddy_net)
            ▼
      ┌───────────┐
      │  Grafana  │ (Dashboard UI - Port 3000)
      └─────▲─────┘
            │
      ┌─────┴──────┐
      │ Prometheus │ (Time-series Database - Port 9090)
      └─────▲──────┘
            │
    ┌───────┴────────┐
    ▼                ▼
┌───────────────┐ ┌───────────────┐
│ Node Exporter │ │ Remote Host   │ (System Metrics - Port 9100)
│ (Local Host)  │ │ (Node Exp.)   │
└───────────────┘ └───────────────┘
```

### Components

- **Node Exporter**: Exposes hardware and OS metrics from the host Linux kernel (`/proc`, `/sys`).
- **Prometheus**: Scrapes metrics from Node Exporter at 5-second intervals and stores them in a persistent volume.
- **Grafana**: Visualizes metrics through customizable dashboards.
- **Caddy (Reverse Proxy)**: Routes traffic securely to Grafana with automatic SSL/TLS certificate management.

---

## Project Structure

```text
.
├── docker-compose.yml     # Container definitions and volume mounts
├── prometheus.yml         # Prometheus scrape targets configuration
├── .env.example           # Template for environment variables
├── .gitignore             # Prevents committing secrets (.env)
└── README.md
```

---

## Quick Start

### 1. Prerequisites

- Docker and Docker Compose plugin installed.
- A shared Docker network for proxying:
  ```bash
  docker network create caddy_net
  ```

### 2. Configuration

Clone the repository and prepare the environment file:

```bash
git clone https://github.com/kicsirigo/monitor-in-docker-compose.git
cd monitor-in-docker-compose
cp .env.example .env
```

Edit `.env` to set your Grafana credentials:

```dotenv
GRAFANA_USER=admin
GRAFANA_PASSWORD=your_secure_password
GRAFANA_PORT=3000
PROMETHEUS_PORT=9090
NODE_EXPORTER_PORT=9100
```

### 3. Deploy the Stack

```bash
docker compose up -d
```

Verify all containers are up and healthy:

```bash
docker compose ps
```

---

## Reverse Proxy Integration (Caddy)

To expose Grafana publicly or on a subdomain with automatic HTTPS, attach your Caddy instance to `caddy_net` and add the following block to your `Caddyfile`:

```caddyfile
grafana.yourdomain.com {
    tls {
        dns duckdns YOUR_DUCKDNS_TOKEN
    }
    reverse_proxy grafana:3000
}
```

Reload Caddy to apply changes:

```bash
docker exec -w /etc/caddy caddy caddy reload
```

---

## Setting up Dashboards in Grafana

1. Navigate to Grafana at `http://localhost:3000` (or via your Caddy domain).
2. Go to **Connections** > **Data Sources** > **Add data source**.
3. Select **Prometheus** and set the Server URL to:
   ```text
   http://prometheus:9090
   ```
4. Click **Save & Test**.
5. Go to **Dashboards** > **New** > **Import**.
6. Enter dashboard ID **`1860`** (*Node Exporter Full*) and click **Load**.
7. Select the Prometheus data source and click **Import**.

---

## Monitoring Additional Machines

To monitor multiple servers from a single Grafana instance:

1. Run `node-exporter` on the target remote host.
2. Edit `prometheus.yml` to include the target's IP:
   ```yaml
   scrape_configs:
     - job_name: 'local-node'
       static_configs:
         - targets: ['node-exporter:9100']

     - job_name: 'remote-node'
       static_configs:
         - targets: ['192.168.100.2:9100']
   ```
3. Restart Prometheus to reload the target list:
   ```bash
   docker compose restart prometheus
   ```

---

## Security Considerations

- Credentials are stored strictly in `.env`, which is ignored by Git.
- Internal ports (9090, 9100) do not need to be published to the host if all access is routed through the internal Docker network.
