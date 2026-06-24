<![CDATA[# 🏗️ ComplianceAI — Infrastructure

> Docker Compose orchestration for the entire AI Regulatory Compliance Agent stack. Manages 6 services (PostgreSQL, Redis, Qdrant, Ingestion Pipeline, Backend API, Frontend) with health checks, persistent volumes, and custom DNS configuration.

[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)](https://docs.docker.com/compose/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16--alpine-4169E1?logo=postgresql&logoColor=white)](https://postgresql.org)
[![Redis](https://img.shields.io/badge/Redis-7--alpine-DC382D?logo=redis&logoColor=white)](https://redis.io)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 📋 Table of Contents

- [Overview](#overview)
- [Services](#services)
- [Quick Start](#quick-start)
- [Environment Variables](#environment-variables)
- [Network Architecture](#network-architecture)
- [Persistent Volumes](#persistent-volumes)
- [DNS Configuration](#dns-configuration)
- [Operational Commands](#operational-commands)
- [Project Structure](#project-structure)
- [License](#license)

---

## Overview

This repo contains the **Docker Compose** configuration that wires together all services of ComplianceAI. It is the single entry point for running the entire application stack locally.

---

## Services

| Service | Image | Port | Purpose | Health Check |
|---------|-------|:----:|---------|:------------:|
| **postgres** | `postgres:16-alpine` | `5432` | User data, company profiles, analysis results, PDF reports | ✅ `pg_isready` |
| **redis** | `redis:7-alpine` | `6379` | Session progress tracking, SSE pub/sub | ✅ `redis-cli ping` |
| **qdrant** | `qdrant/qdrant:latest` | `6333`, `6334` | Vector database for regulation embeddings | — |
| **ingestion** | Custom (Python 3.11) | — | One-shot pipeline: download → extract → chunk → embed → store | Exits on completion |
| **backend** | Custom (Python 3.11) | `8000` | FastAPI API + LangGraph agent pipeline | — |
| **frontend** | Custom (Node 20) | `5173` | React + Vite dashboard | — |

### Service Dependencies

```
                 ┌──────────┐
                 │ postgres │ ← healthcheck: pg_isready
                 └────┬─────┘
                      │
                 ┌────▼─────┐
                 │  redis   │ ← healthcheck: redis-cli ping
                 └────┬─────┘
                      │
┌──────────┐     ┌────▼─────┐
│  qdrant  │◄────┤ backend  │ ← depends on: postgres (healthy),
└────┬─────┘     └────┬─────┘   redis (healthy), qdrant (started)
     │                │
     │           ┌────▼─────┐
     │           │ frontend │ ← depends on: backend
     │           └──────────┘
     │
┌────▼──────┐
│ ingestion │ ← depends on: qdrant (started)
└───────────┘   restart: "no" (runs once)
```

---

## Quick Start

### 1. Clone all repositories

```bash
# Clone into the expected directory structure
git clone <org>/infrastructure Infrastructure
git clone <org>/backend backend
git clone <org>/frontend frontend
git clone <org>/ingestion ingestion
```

### 2. Configure environment

```bash
cd Infrastructure
cp .env.example .env
# Edit .env with your API keys and credentials
```

### 3. Place regulation PDFs

Place government regulation PDF files into `data/raw/`:

```bash
mkdir -p ../data/raw
# Copy your regulation PDFs here
```

### 4. Launch the stack

```bash
# Build and start all services
docker compose up --build

# Or in detached mode
docker compose up --build -d
```

### 5. Access the application

| Service | URL |
|---------|-----|
| **Frontend** | http://localhost:5173 |
| **Backend API** | http://localhost:8000 |
| **API Docs** | http://localhost:8000/docs |
| **Qdrant Dashboard** | http://localhost:6333/dashboard |

---

## Environment Variables

Create a `.env` file from `.env.example`:

```env
# LLM
GEMINI_API_KEY=your-gemini-api-key-here

# PostgreSQL
POSTGRES_USER=complianceai
POSTGRES_PASSWORD=your-secure-password
POSTGRES_DB=compliance_db
POSTGRES_HOST=postgres
POSTGRES_PORT=5432

# Redis
REDIS_HOST=redis
REDIS_PORT=6379

# Qdrant
QDRANT_HOST=qdrant
QDRANT_PORT=6333
QDRANT_COLLECTION=regulations

# Embedding
EMBEDDING_MODEL=all-MiniLM-L6-v2
CHUNK_SIZE=500
CHUNK_OVERLAP=50

# Auth
JWT_SECRET_KEY=your-jwt-secret-key-here
JWT_ALGORITHM=HS256
JWT_EXPIRE_MINUTES=1440
```

> ⚠️ **Never commit the `.env` file.** It contains secrets and API keys. Only `.env.example` is version-controlled.

---

## Network Architecture

All services communicate over a custom Docker bridge network: `compliance_network`.

```
┌─────────────────────────────────────────────────┐
│              compliance_network (bridge)         │
│                                                  │
│  postgres:5432  redis:6379  qdrant:6333/6334    │
│  backend:8000   frontend:5173  ingestion        │
│                                                  │
│  DNS: 8.8.8.8, 8.8.4.4 (for backend/ingestion) │
└─────────────────────────────────────────────────┘
```

### External Port Mapping

| Host Port | Container | Service |
|:---------:|-----------|---------|
| `5432` | `postgres:5432` | PostgreSQL |
| `6379` | `redis:6379` | Redis |
| `6333` | `qdrant:6333` | Qdrant REST API |
| `6334` | `qdrant:6334` | Qdrant gRPC |
| `8000` | `backend:8000` | FastAPI Backend |
| `5173` | `frontend:5173` | Vite Dev Server |

---

## Persistent Volumes

| Volume | Mount Point | Purpose |
|--------|-------------|---------|
| `postgres_data` | `/var/lib/postgresql/data` | PostgreSQL database files |
| `redis_data` | `/data` | Redis persistence |
| `qdrant_data` | `/qdrant/storage` | Qdrant vector storage |
| `../data` (bind) | `/app/data` | Shared data directory (PDFs, extracted text, chunks) |

---

## DNS Configuration

The `backend` and `ingestion` services use custom DNS to avoid resolution issues in Docker:

- **DNS Servers**: `8.8.8.8`, `8.8.4.4` (Google Public DNS)
- **Custom resolv.conf**: Mounted from `dns/resolv.conf`
- **dns_search**: Empty (prevents appending search domains)

This is necessary because the ingestion pipeline and backend (in external mode) make outbound HTTP requests that require reliable DNS resolution.

---

## Operational Commands

```bash
# Start everything
docker compose up --build -d

# View logs
docker compose logs -f backend
docker compose logs -f ingestion

# Restart a specific service
docker compose restart backend

# Stop everything
docker compose down

# Stop and remove volumes (⚠️ deletes all data)
docker compose down -v

# Rebuild a specific service
docker compose up --build backend

# Run ingestion pipeline manually
docker compose run --rm ingestion

# Check service health
docker compose ps
```

---

## Project Structure

```
Infrastructure/
├── .env                    # Environment variables (not committed)
├── .env.example            # Template for .env
├── .gitignore              # Ignores .env, volumes, etc.
├── docker-compose.yml      # Full stack orchestration
├── LICENSE                 # MIT License
│
└── dns/
    └── resolv.conf         # Custom DNS resolver config
```

---

## License

This project is licensed under the **MIT License** — see the [LICENSE](LICENSE) file for details.
]]>