# DeployReady - Kora API DevOps Implementation

This repository contains my implementation of the DeployReady challenge for Kora Analytics.

I containerized the Node.js API, created an automated CI/CD workflow with GitHub Actions, pushed the image to GitHub Container Registry, and deployed the service to AWS EC2.

## Architecture Overview

Deployment flow:

1. Developer pushes code to `main`.
2. GitHub Actions runs tests in `app/`.
3. Workflow builds Docker image tagged with commit SHA and `latest`.
4. Workflow pushes image to GitHub Container Registry.
5. Workflow SSHes into EC2, pulls the latest image, and restarts the `kora-api` container.
6. The service is exposed publicly on EC2 port `80`.

## Application Endpoints

The API exposes:

- `GET /health` returns `{"status":"ok"}`
- `GET /metrics` returns uptime and memory usage
- `POST /data` echoes a JSON payload

## Local Development

Run locally without Docker:

```bash
cd app
npm install
npm start
```

Run with Docker Compose:

```bash
docker compose up --build
```

Health check:

```bash
curl http://localhost:3000/health
```

## Containerization

- `Dockerfile` runs the app on Node 18 Alpine.
- The container runs as a non-root user.
- The app reads `PORT` from the environment.
- `docker-compose.yml` maps `3000:3000` for local usage.
- `.env.example` is included with placeholder values.

## CI/CD Pipeline

Workflow file: `.github/workflows/deploy.yml`

Pipeline order on push to `main`:

1. Install dependencies
2. Run tests (`npm test`)
3. Build image tagged with `${{ github.sha }}` and `latest`
4. Login and push to GHCR
5. Deploy on EC2 over SSH

Required secrets:

- `GHCR_USERNAME`
- `GHCR_TOKEN`
- `EC2_HOST`
- `EC2_USER`
- `EC2_SSH_KEY`

## AWS Deployment Summary

- Region: `eu-north-1`
- EC2 Public IP: `16.171.134.13`
- EC2 OS: Amazon Linux 2023
- Security Group:
  - HTTP `80` from `0.0.0.0/0`
  - SSH `22` from my IP only (`/32`)

Public health check:

```bash
curl http://16.171.134.13/health
```

Expected response:

```json
{"status":"ok"}
```

## Decisions and Notes

- Used GHCR for image hosting to keep CI/CD integrated with GitHub.
- Used Docker on EC2 for simple and repeatable deployment.
- Kept the application logic unchanged and focused on DevOps requirements.

## Problems Faced and How I Solved Them

### 1. `GET /data` did not work

I first tested `/data` with a `GET` request and got `Cannot GET /data`.
After checking the application contract, I confirmed that `/data` only supports `POST`.
I then retried it with a JSON body using a `POST` request, which returned the expected response.

Correct test:

```bash
curl -X POST http://localhost:3000/data \
  -H "Content-Type: application/json" \
  -d '{"name":"test"}'
```

### 2. PowerShell treated `curl` differently

In PowerShell, `curl` is not always the same as Linux `curl`.
My multiline command split `-d` into a separate line, so PowerShell treated it like a new command.
I fixed it by using `Invoke-RestMethod` with `-Method Post`, which worked correctly.

PowerShell-safe test:

```powershell
Invoke-RestMethod -Method Post -Uri http://localhost:3000/data -ContentType "application/json" -Body '{"name":"test"}'
```

### 3. GHCR login scope issue

At first, Docker push to GHCR failed because the token did not have the right package scopes.
I created a new classic GitHub PAT with the required package permissions and then the push succeeded.

### 4. SSH access changed because my IP changed

My SSH rule needed to match my current public IP.
I updated the Security Group inbound rule to allow SSH from my current `/32` IP only, which let me connect to EC2 securely.

Detailed AWS setup and operations are documented in `DEPLOYMENT.md`.
