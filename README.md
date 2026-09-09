# Self-Hosted Monitoring Stack with Docker Compose

A complete, production-ready monitoring setup based on Docker Compose. This stack collects, stores, and visualizes system metrics (CPU, memory, disk, network) across multiple Linux hosts with automatic HTTPS provided by Caddy and DuckDNS.

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
┌───────────────┐ ┌──────────────────┐
│ Node Exporter │ │ Remote Host      │ (System Metrics - Port 9100)
│ (Local Host)  │ │ (e.g. ArchLinux) │
└───────────────┘ └──────────────────┘
```

### Components

- **Node Exporter**: Exposes hardware and OS metrics from the host Linux kernel (`/proc`, `/sys`). Configured with filters to exclude virtual Docker veth/bridge network interfaces from metric noise.
- **Prometheus**: Scrapes metrics from local and remote Node Exporter instances at 5-second intervals and stores them in a persistent volume.
- **Grafana**: Visualizes metrics through customizable dashboards. Accessible securely via Caddy reverse proxy.
- **Caddy (Reverse Proxy)**: Routes traffic securely to Grafana with automated Let's Encrypt / ZeroSSL TLS certificates via DuckDNS DNS-01 challenge.

---

## Project Structure

```text
.
├── docker-compose.yml     # Monitoring stack (Grafana, Prometheus, Node Exporter)
├── prometheus.yml         # Prometheus scrape targets configuration
├── .env.example           # Template for monitoring environment variables
├── .gitignore             # Ignores sensitive environment files
├── caddy/                 # Caddy reverse proxy stack with DuckDNS
│   ├── docker-compose.yml # Caddy container definition
│   ├── Caddyfile          # Reverse proxy configuration
│   ├── .env.example       # Template for Caddy/DuckDNS credentials
│   └── .gitignore
└── README.md
```

---

## Quick Start

### 1. Prerequisites

- Docker and Docker Compose installed.
- Create the shared external Docker network used by Caddy and Grafana:
  ```bash
  docker network create caddy_net
  ```

### 2. Deploy Monitoring Stack

1. Copy the environment configuration template:
   ```bash
   cp .env.example .env
   ```

2. Customize `.env` credentials and ports:
   ```dotenv
   # Grafana Admin Credentials
   GRAFANA_USER=admin
   GRAFANA_PASSWORD=your_secure_password
   GRAFANA_PORT=3000

   # Service Ports
   PROMETHEUS_PORT=9090
   NODE_EXPORTER_PORT=9100
   ```

3. Launch the monitoring stack:
   ```bash
   docker compose up -d
   ```

4. Verify all containers are running:
   ```bash
   docker compose ps
   ```

---

### 3. Deploy Reverse Proxy (Caddy with DuckDNS)

A standalone Caddy setup with DuckDNS plugin (`ghcr.io/serfriz/caddy-duckdns`) is included in the `caddy/` directory:

1. Navigate to the Caddy folder:
   ```bash
   cd caddy
   cp .env.example .env
   ```

2. Fill in your DuckDNS domain, token, and ACME notification email in `caddy/.env`:
   ```dotenv
   ACME_EMAIL=your_email@example.com
   GRAFANA_DOMAIN=grafana.yourdomain.duckdns.org
   DUCKDNS_API_TOKEN=your_duckdns_api_token
   ```

3. Start Caddy:
   ```bash
   docker compose up -d
   ```

> If you already have an existing Caddy instance running on `caddy_net`, you can instead add the following block to your existing `Caddyfile`:
> ```caddyfile
> grafana.yourdomain.duckdns.org {
>     tls {
>         dns duckdns YOUR_DUCKDNS_TOKEN
>     }
>     reverse_proxy grafana:3000
> }
> ```
> And reload Caddy:
> ```bash
> docker exec -w /etc/caddy caddy caddy reload
> ```

---

## Setting up Dashboards in Grafana

1. Open Grafana in your browser (at `http://localhost:3000` or `https://grafana.yourdomain.duckdns.org`).
2. Log in with the credentials defined in `.env`.
3. Add Prometheus data source:
   - Navigate to **Connections** > **Data Sources** > **Add data source**.
   - Select **Prometheus**.
   - Set URL to `http://prometheus:9090`.
   - Click **Save & Test**.
4. Import Node Exporter Dashboard:
   - Go to **Dashboards** > **New** > **Import**.
   - Enter Dashboard ID **`1860`** (*Node Exporter Full*) and click **Load**.
   - Select the Prometheus data source and click **Import**.

---

## Monitoring Additional Hosts

To monitor multiple servers from this single Prometheus & Grafana stack:

1. Install and run `node-exporter` on each remote host (e.g. ArchLinux, Raspberry Pi, etc.).
2. Edit `prometheus.yml` to add your remote targets:
   ```yaml
   global:
     scrape_interval: 5s

   scrape_configs:
     - job_name: 'localhost'
       static_configs:
         - targets: ['node-exporter:9100']

     - job_name: 'archlinux'
       static_configs:
         - targets: ['192.168.100.2:9100']
   ```
3. Reload Prometheus without restarting:
   ```bash
   docker compose exec prometheus kill -HUP 1
   # Or restart the container:
   docker compose restart prometheus
   ```

---

## Key Configuration Details

### Virtual NIC Metric Noise Filter
Node Exporter includes collector flags to exclude dynamic Docker virtual Ethernet (`veth*`), bridge (`br-*`), and `docker*` interfaces from cluttering Grafana dashboards:
```yaml
command:
  - '--path.procfs=/host/proc'
  - '--path.sysfs=/host/sys'
  - '--path.rootfs=/rootfs'
  - '--collector.netdev.device-exclude=^(veth.*|br-.*|docker.*)$'
  - '--collector.netclass.ignored-devices=^(veth.*|br-.*|docker.*)$'
```

### Security Considerations
- `.env` files are ignored by git to keep your DuckDNS tokens and passwords safe.
- Node Exporter port `9100` does not need to be published to the host network because Prometheus queries it directly via internal Docker networking (`node-exporter:9100`).
- Grafana traffic is protected by HTTPS with automatic certificate renewal via Caddy and DuckDNS.
