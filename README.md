# LU 4.7 — Fixing Environment Variables in Cloud Deployments

## Overview
This repository is an assessment for debugging and fixing environment variable configurations in a containerized Node.js application.

The application currently fails to start correctly because one or more configuration issues exist. 
Use logs and environment inspection to diagnose the problem.

## Setup & Reproduction Instructions

### 1. Build the Docker Image
```bash
docker build -t envlab .
```

### 2. Observe the Initial Failure (Before Fix)
```bash
docker run --name app envlab
docker logs app
```
*Output: `Error: DATABASE_URL is undefined` (Container exits with code 1)*

---

## The Proper Runtime Fix

Configuration and secrets must be separated and injected at runtime rather than baked into the container image:

1. **Non-secret configuration** is placed in a git-ignored `.env` file and passed using `--env-file .env`:
   ```bash
   cp .env.example .env
   ```
   Edit `.env` with non-sensitive configs:
   ```env
   DATABASE_URL=postgres://user:password@localhost:5432/mydb
   LOG_LEVEL=info
   ```

2. **Secrets (API Keys, tokens)** are injected at runtime using `-e` / CLI environment variables, never committed to Git or baked into the Dockerfile:
   ```bash
   # On Linux/macOS
   export API_KEY="my-secret-api-key"
   docker run -d --env-file .env -e API_KEY=$API_KEY -p 3000:3000 --name app-fixed envlab

   # On Windows PowerShell
   $env:API_KEY="my-secret-api-key"
   docker run -d --env-file .env -e API_KEY=$env:API_KEY -p 3000:3000 --name app-fixed envlab
   ```

---

## Verification

### 1. Verify Clean Startup Logs
```bash
docker logs app-fixed
```
*Expected output:*
```
Starting app with log level: info
Server is running on port 3000
```

### 2. Inspect Injected Environment Variables
```bash
docker exec app-fixed printenv
```
Confirm `DATABASE_URL`, `LOG_LEVEL`, and `API_KEY` are present and accurately populated.

### 3. Verify Health Check Endpoint
```bash
curl http://localhost:3000/health
# Response: {"status":"ok"}
```

### 4. Verify Image Layers (No Secrets Baked In)
```bash
docker history envlab
```
Confirms neither `API_KEY` nor hardcoded env overrides are baked into the image layers.

---

## Cloud Deployment Mapping
Refer to [`CLOUD_RUN.md`](CLOUD_RUN.md) and [`docs/cloud-run-template.md`](docs/cloud-run-template.md) for how these runtime mechanisms map to Google Cloud Run (`--set-env-vars` and `--set-secrets`).
