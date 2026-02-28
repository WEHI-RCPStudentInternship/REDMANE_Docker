# REDMANE Docker Deployment

This repository contains the Docker orchestration files (docker-compose and dockerfiles) for deploying the REDMANE Data Registry application stack.

## Overview

REDMANE (Research Data Management and Analysis Environment) is a web application that tracks research datasets, project data, etc. without actually storing them. This repo provides the top-level Docker configuration that ties together the frontend, backend, and database services with SSL termination via https-portal container (instead of separate nginx and certbot instances).

## Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│  Docker Network (redmane-network)                               │
│                                                                 │
│  ┌──────────────────┐                                           │
│  │  frontend-build  │                                           │
│  │  (Node.js)       │──► Builds React static files once,        │
│  │                  │    copies to shared volume, then exits    │
│  └──────────────────┘                                           │
│           │ frontend_static volume                              │
│           ▼                                                     │
│  ┌─────────────┐          ┌─────────────┐                       │
│  │ https-portal│          │  backend    │                       │
│  │  :80/:443   │─────────►│  (FastAPI)  │                       │
│  │             │ /fastapi/│  :8888      │                       │
│  │ Serves      │          └─────────────┘                       │
│  │ static      │                │                               │
│  │ React files │                │                               │
│  │ at /        │                ▼                               │
│  └─────────────┘          ┌─────────────┐                       │
│                           │     db      │                       │
│                           │ (PostgreSQL)│                       │
│                           │  :5432      │                       │
│                           └─────────────┘                       │
│                                                                 │
│  Path-based routing:                                            │
│    /           → Static React files (served by https-portal)    │
│    /fastapi/   → backend:8888                                   │
└─────────────────────────────────────────────────────────────────┘
```

## Repository Structure
```
REDMANE_Docker/
├── docker-compose.yaml              # Main orchestration file
├── redmane_fastapi.dockerfile       # Backend container build
├── redmane_reactjs.dockerfile       # Frontend multi-stage build (build React, copy static files)
├── data-registry.wehi-rcp.cloud.edu.au.conf.erb  # Custom nginx config for path-based routing
├── .dockerignore                    # Excludes unnecessary files from Docker build context
├── .gitignore                       # Excludes files/folders that may cause conflict when cloning
└── README.md
```

**Note:** The following folders are cloned separately and/or excluded via .gitignore:
- `REDMANE_fastapi/` - Backend application (separate repo)
- `REDMANE_react.js/` - Frontend application (separate repo)
- `backups/` - VM's backup folder (not currently tracked in any git repo)

## Related Repositories

- [REDMANE_fastapi](https://github.com/WEHI-RCPStudentInternship/REDMANE_fastapi) - FastAPI backend
- [REDMANE_react.js](https://github.com/WEHI-RCPStudentInternship/REDMANE_react.js) - React frontend
- [REDMANE_fastapi_public_data](https://github.com/WEHI-RCPStudentInternship/REDMANE_fastapi_public_data) - Database initialization scripts

## Prerequisites

- Docker and Docker Compose installed (done)
- Access to the target VM (via SSH, check technical diaries)
- Domain DNS configured to point to the VM's IP address (done)

## Context on https-portal

It is a ready-to-use docker image that combines the functionality of nginx (reverse proxying, hosting static frontend files, SSL termination) and certbot (automated SSL certificate obtainment and renewal/management). For more context, visit: https://github.com/SteveLTN/https-portal

## Deployment Steps

### 1. Clone this repository on the REDMANE folder of the Data Registry VM
```bash
cd REDMANE
git clone https://github.com/WEHI-RCPStudentInternship/REDMANE_Docker.git .
```

### 2. Clone the application repositories (please change accordingly if newer branches are made)
```bash
# Frontend
git clone -b 13-semester2_2025 https://github.com/WEHI-RCPStudentInternship/REDMANE_react.js.git

# Backend
git clone -b 13-2025-Semester-2 https://github.com/WEHI-RCPStudentInternship/REDMANE_fastapi.git

# Database initialisation scripts
git clone -b sem_2_2025 https://github.com/WEHI-RCPStudentInternship/REDMANE_fastapi_public_data.git REDMANE_fastapi/data/REDMANE_fastapi_public_data
```

### 3. Configure environment

Before starting, verify the following in `docker-compose.yaml`:
- `STAGE` is set appropriately (`local`, `staging`, or `production`)
  - Do not use 'production' unless the setup has been successfully tested in staging. 'production' will hit the production side of Let's Encrypt and they have rate limits
- Database credentials match other parts of the backend and database codes
- Domain name in https-portal container's DOMAINS variable matches your setup

### 4. Start the stack
```bash
docker compose up --build -d
```

The `frontend-build` container will build the React application, copy the static files to a shared volume, and then exit. This is expected — `docker ps -a` will show it as "Exited (0)".

### 5. Verify deployment

- Frontend: access `https://data-registry.wehi-rcp.cloud.edu.au/projects` on an incognito browser (so that caches are not saved and you can test builds more accurately)
- Backend API: perform this on your terminal:
```bash
curl https://data-registry.wehi-rcp.cloud.edu.au/projects # if SSL certs have been obtained
curl -k https://data-registry.wehi-rcp.cloud.edu.au/projects # in local/staging STAGE
```

## Configuration Details

### https-portal Stages

| Stage | Purpose | Certificates |
|-------|---------|--------------|
| `local` | Local development | Self-signed |
| `staging` | Testing SSL setup | Let's Encrypt staging (not trusted) |
| `production` | Live deployment | Let's Encrypt production (trusted) |

**Important:** Always test with `staging` before switching to `production` to avoid hitting Let's Encrypt rate limits.

### Path-Based Routing

The `.conf.erb` file configures nginx (inside https-portal) to:
- Serve static React files at `/` directly from the shared volume (with `try_files` fallback to `index.html` for client-side routing)
- Proxy all requests to `/fastapi/` → FastAPI backend container

### Frontend Multi-Stage Build

The `redmane_reactjs.dockerfile` uses a two-stage Docker build:
1. **Stage 1 (Node.js):** Installs dependencies and runs `npm run build` to produce static files in `dist/`
2. **Stage 2 (Alpine):** Copies only the built `dist/` files into a lightweight image (~5MB), discarding Node.js and node_modules

At runtime, the container copies the static files to a shared Docker volume (`frontend_static`) that https-portal serves directly. The container then exits.

## Troubleshooting

### Check container logs
```bash
docker compose logs -f [service_name]
```

### Rebuild after changes
```bash
docker compose down
docker compose up --build -d
```

### Reset the database to default seed data

There are two methods to reset the database.

**Method 1: Run the reset script (recommended — no downtime for containers)**
```bash
bash reset_database.sh
```
This script drops the `public` schema, recreates it, and re-runs the init SQL file, all while keeping the Docker containers running. See `reset_database.sh` for details on what it does.

**Method 2: Remove the PostgreSQL volume (nuclear option — requires restarting the stack)**
```bash
# Backup current database first (optional but recommended)
docker exec redmane-db pg_dump -U postgres readmedatabase > ~/REDMANE/backups/database_backup_$(date +%Y%m%d_%H%M%S).sql

# Stop all containers
docker compose down

# Remove the postgres volume to trigger re-initialisation
docker volume rm redmane_postgres_data

# Bring everything back up
docker compose up -d
```
This forces PostgreSQL to re-run the init scripts on startup since the data volume is empty.


# REDMANE Auth0 Setup Guide

This app demonstrates how to use the Auth0 SDK for authentication.

---

## Getting Started

### 1. Clone the Repositories

```bash
# Clone the Docker repo
git clone <REDMANE_Docker_repo_url>
cd REDMANE_Docker
git checkout Auth0

# Clone the React frontend
git clone <REDMANE_react.js_repo_url>
cd REDMANE_react.js
git checkout Auth0

# Clone the FastAPI backend
git clone <REDMANE_fastapi_repo_url>
cd REDMANE_fastapi
git checkout Auth0
```

> **Note:** The current repo is configured for an existing Auth0 application. If you want to use it as-is, skip to [Run the Application](#run-the-application). To set up your own Auth0 application, follow the steps below.

---

## How to Make an Auth0 App

### 2. Create an Auth0 Account and Application

1. Create an account at [manage.auth0.com](https://manage.auth0.com)
2. Navigate to **Applications → Applications → Create Application**
3. Set:
   - **Name**: `REDMANE` (or any name)
   - **Type**: Single Page Application
4. Under **Settings**, configure:
   - **Allowed Callback URLs**:
     ```
     http://localhost:5173, http://localhost:4173, https://your-production-domain.com
     ```
   - **Allowed Logout URLs**:
     ```
     http://localhost:5173/login, http://localhost:4173/login, https://your-production-domain.com/login
     ```
   - **Allowed Web Origins**:
     ```
     http://localhost:5173, http://localhost:4173, https://your-production-domain.com
     ```
5. Click **Save Changes** and note down your **Domain** and **Client ID**

### 3. Create an Auth0 API

1. Navigate to **Applications → APIs → Create API**
2. Set:
   - **Name**: `REDMANE API`
   - **Identifier**: `redmane`
   - **Signing Algorithm**: RS256
3. Click **Create**
4. Navigate to **Applications → Applications → your application → APIs tab**
5. Toggle on **REDMANE API** to grant your application access

---

## Configure the React Frontend

### 4. Create Environment Files

Create `.env.development` in `REDMANE_react.js/`:

```bash
VITE_API_BASE_URL=http://localhost:8888
VITE_AUTH0_DOMAIN=        # paste your domain here
VITE_AUTH0_CLIENT_ID=     # paste your client ID here
VITE_AUTH0_AUDIENCE=redmane
```

Create `.env.production` in `REDMANE_react.js/`:

```bash
VITE_API_BASE_URL=https://your-production-domain.com/fastapi
VITE_AUTH0_DOMAIN=        # paste your domain here
VITE_AUTH0_CLIENT_ID=     # paste your client ID here
VITE_AUTH0_AUDIENCE=redmane
```

---

## Configure the FastAPI Backend

### 5. Update `auth.py`

**For development** (with fallback defaults):

```python
AUTH0_DOMAIN = os.getenv("AUTH0_DOMAIN", "insert-domain-here").strip()
AUTH0_AUDIENCE = os.getenv("AUTH0_AUDIENCE", "redmane").strip()
```

**For production** (fails loudly if env vars are missing):

```python
AUTH0_DOMAIN = os.getenv("AUTH0_DOMAIN", "").strip()
AUTH0_AUDIENCE = os.getenv("AUTH0_AUDIENCE", "").strip()
AUTH0_ISSUER = f"https://{AUTH0_DOMAIN}/"

if not AUTH0_DOMAIN:
    raise RuntimeError("AUTH0_DOMAIN environment variable is not set")
if not AUTH0_AUDIENCE:
    raise RuntimeError("AUTH0_AUDIENCE environment variable is not set")
```

---

## Configure Docker

### 6. Update `docker-compose.yaml`

Under the backend service, set the Auth0 environment variables:

```yaml
backend:
  environment:
    - AUTH0_DOMAIN=${AUTH0_DOMAIN}
    - AUTH0_AUDIENCE=${AUTH0_AUDIENCE}
```

### 7. Create a `.env` File in `REDMANE_Docker/`

> Unlike React, Docker Compose only reads a single `.env` file — not `.env.development` or `.env.production`.

**For local development:**

```bash
AUTH0_DOMAIN=        # insert your Auth0 domain
AUTH0_AUDIENCE=redmane
DOMAIN=localhost
STAGE=local
```

**For production (on the server):**

```bash
AUTH0_DOMAIN=        # insert your Auth0 domain
AUTH0_AUDIENCE=redmane
DOMAIN=data-registry.wehi-rcp.cloud.edu.au
STAGE=production
```

### 8. Add `.env` to `.gitignore`

```bash
echo ".env" >> .gitignore
```

---

## Run the Application

### 9. Start the Backend and Database

```bash
cd REDMANE_Docker
docker compose up -d
```

### 10. Load the Database (First Time Only)

```bash
docker exec -it redmane-db psql -U postgres -d readmedatabase -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"
docker exec -i redmane-db psql -U postgres -d readmedatabase < REDMANE_fastapi/data/REDMANE_fastapi_public_data/readmedatabase.sql
```

### 11. Start the Frontend

**Development:**

```bash
cd REDMANE_react.js
npm install
npm run dev
```

**Production build:**

```bash
cd REDMANE_react.js
npm run build
npm run preview  # to test the production build locally
```

---

## Stopping the Application

```bash
cd REDMANE_Docker
docker compose down
```

---

## Debugging

View backend logs:

```bash
docker logs redmane_docker-backend-1
```

---
