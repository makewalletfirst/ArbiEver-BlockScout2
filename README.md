# ArbiEver Blockscout

[![CI Status](https://github.com/makewalletfirst/ArbiEver-BlockScout/actions/workflows/ci.yml/badge.svg)](https://github.com/makewalletfirst/ArbiEver-BlockScout/actions)
[![Network: ArbiEver](https://img.shields.io/badge/Network-ArbiEver-blueviolet?style=flat-square)](https://arbiever.ever-chain.xyz)
[![Chain ID: 580511](https://img.shields.io/badge/Chain_ID-580511-orange?style=flat-square)](https://arbiever.ever-chain.xyz)
[![L2 Mode: AnyTrust](https://img.shields.io/badge/L2_Mode-AnyTrust-blue?style=flat-square)](https://arbiever.ever-chain.xyz)

This repository hosts the deployment configurations and custom proxy definitions for **ArbiEver Blockscout**—a fully-customized, full-stack blockchain explorer dedicated specifically to the **ArbiEver L2** chain. 

ArbiEver Blockscout is built upon a highly optimized fork of [Blockscout](https://github.com/blockscout/blockscout), packaged into a lightweight, 5-container minimal configuration tailored to run efficiently on infrastructure with 4 CPU cores and 12GB of RAM.

---

## 🌐 ArbiEver L2 Chain Context

**ArbiEver** is a custom Layer 2 (L2) rollup built using the **Arbitrum Nitro AnyTrust** technology stack. It operates directly on top of the parent Layer 1 (L1) chain, **EtherEver**.

### Core Chain Specifications

| Specification | Layer 1 (Parent Chain) | Layer 2 (Rollup Chain) |
| :--- | :--- | :--- |
| **Name** | EtherEver | **ArbiEver** |
| **Chain ID** | `58051` (0xe2c3) | **`580511`** (0x8dba7) |
| **EVM Compatibility** | Pre-London (No `PUSH0` opcode support) | Custom Nitro execution (Shanghai compatible target) |
| **Gas Model** | Legacy Gas Model (EIP-1559 Unsupported) | Optimistic custom gas target with Brotli L1 batch compression |
| **Data Availability** | On-chain | **AnyTrust DAC Mode** (Single-member committee via `daserver`) |
| **Public RPC** | `https://rpc-ether.ever-chain.xyz` | Connected locally to Sequencer RPC (`:8449`) |

### Why AnyTrust & How It Links
Instead of posting transaction raw data directly onto EtherEver L1 (which would be extremely gas-prohibitive given the L1's 30M gas limit and legacy gas model), ArbiEver utilizes the **AnyTrust** protocol. 
1. **Data Availability Committee (DAC)**: The DAC server (`daserver`) stores transaction data off-chain and guarantees its availability.
2. **Sequencer & Batch Poster**: The Sequencer processes transactions, and the Batch Poster compresses L2 batches (using Brotli) and posts lightweight data availability certificates (`keysets`) to the L1 `SequencerInbox`.
3. **Validator / Staker**: Independent validators verify assertions and secure the chain, operating with a 7-day optimistic challenge period.

---

## 🎨 ArbiEver Custom Branded Fork

ArbiEver Blockscout features a dedicated brand identity configured directly in the frontend and proxy configurations:
- **Network Name**: `ArbiEver`
- **Native Currency Symbol**: `ETE` (hard-pegged to parent chain EtherEver ETE)
- **Visual Branding**: Integrated custom brand icon and favicon (`arbiicon512.png` built directly into `/static/arbiicon.png`).
- **Default Theme**: Sleek Light Mode.
- **Production Grade**: Testnet badge disabled.

---

## 🏗️ 5-Container Architecture

To maintain high performance with a minimal memory footprint, resource-heavy features (like the visualizer, statistics aggregator, signature provider, and user-ops indexer) are disabled, concentrating memory on the core indexer and frontend engine.

```
                  ┌──────────────────────────────┐
                  │   Cloudflare HTTPS Gateway   │
                  │ (arbiever.ever-chain.xyz:443)│
                  └──────────────┬───────────────┘
                                 │ OCI VPN Tunnel
                  ┌──────────────▼───────────────┐
                  │        OCI stunnel           │
                  │    (Forward to port 4005)    │
                  └──────────────┬───────────────┘
                                 │
                 ┌───────────────▼───────────────┐
                 │    arbiever-bs-proxy          │  (Nginx Proxy)
                 │    (Serves /static/ & routes) │
                 └──────┬─────────────────┬──────┘
                        │ /api/           │ /
                        │                 │
     ┌──────────────────▼────────┐ ┌──────▼──────────────────┐
     │ arbiever-bs-backend (4000)│ │arbiever-bs-frontend(3000)│
     │  (Elixir Indexer & API)   │ │      (Next.js UI)       │
     └──────────┬───────────┬────┘ └─────────────────────────┘
                │           │
     ┌──────────▼─────┐  ┌──▼────────────────────────┐
     │arbiever-bs-redis│  │arbiever-bs-db (PostgreSQL)│
     │ (Indexer Cache)│  │    (Chowned by db-init)   │
     └────────────────┘  └───────────────────────────┘
```

| Container | Image | Description / Role |
| :--- | :--- | :--- |
| `arbiever-bs-redis` | `redis:alpine` | Key-value store for Elixir indexer caching. |
| `arbiever-bs-db-init` | `postgres:17` | Standard initialization script container to securely set data folder permissions (`chown 2000:2000` for DB volume). |
| `arbiever-bs-db` | `postgres:17` | High-performance relational database containing blockchain ledger state index. |
| `arbiever-bs-backend` | `blockscout/blockscout:6.10.1` | Main Elixir indexer responsible for scraping block headers, executing release migrations, and running the HTTP API. |
| `arbiever-bs-frontend` | `ghcr.io/blockscout/frontend:v1.36.4` | Next.js server delivering the responsive user interface. |
| `arbiever-bs-proxy` | `silverruler/arbiever-blockscout:latest` | Custom Nginx server routing `/` to the frontend and `/api` to the backend. Serves custom assets like the ArbiEver icon. |

---

## ⚡ Quick Start & Deployment

Follow the steps below to spawn the entire stack using Docker Compose.

### Step 1: Initialize the Environment Config
Create a secure `.env` file containing unique passwords and security keys:
```bash
cat > .env <<EOF
POSTGRES_PASSWORD=$(openssl rand -hex 16)
SECRET_KEY_BASE=$(openssl rand -hex 32)
EOF
chmod 600 .env
```

### Step 2: Spin Up the Stack
Run Docker Compose in detached mode. This loads all customized environment variables from `backend.env` and `frontend.env`:
```bash
docker compose --env-file .env up -d
```
*Note: Containers are configured with `restart: unless-stopped` to survive system reboots.*

### Step 3: Validate the Service
Wait a few seconds for the migrations to complete, and verify that the backend API is successfully responding to queries:
```bash
curl http://localhost:4005/api/v2/stats
```

---

## 🔌 L2 RPC Configuration

The Blockscout backend connects directly to the local Arbitrum Nitro L2 RPC node via:
```ini
ETHEREUM_JSONRPC_HTTP_URL=http://host.docker.internal:8449/
```
The Docker network references `host.docker.internal:host-gateway` to allow backend containers to communicate with the host operating system's Nitro RPC listener binded on `0.0.0.0:8449`.

---

## 🛠️ Local Proxy Image Build (Optional)

If you wish to modify static assets or Nginx routing configurations, you can build the proxy image locally rather than pulling the pre-built image from Docker Hub.

1. **Build the image**:
   ```bash
   docker build -f Dockerfile.proxy -t arbiever-blockscout-proxy:local .
   ```
2. **Update docker-compose**:
   Open `docker-compose.yml` and replace:
   ```yaml
   image: silverruler/arbiever-blockscout:latest
   ```
   with:
   ```yaml
   image: arbiever-blockscout-proxy:local
   ```
3. **Restart the proxy service**:
   ```bash
   docker compose --env-file .env up -d proxy
   ```

---

## 🔗 Related Repositories

- **ArbiEver L2 Core Setup**: [makewalletfirst/ArbiEver](https://github.com/makewalletfirst/ArbiEver) — Scripts, configurations, and playbooks used to bootstrap the Nitro AnyTrust L2.
- **ArbiEver Alethio Explorer**: [makewalletfirst/ArbiEver-Explorer](https://github.com/makewalletfirst/ArbiEver-Explorer) — A lightweight, client-side Alethio explorer used during Phase 8 bootstrap validation.
