# DeployReady - AWS Deployment Notes

This document describes how the Kora API was deployed to AWS EC2 using Docker and GitHub Actions.

## 1. EC2 Setup

- Instance type: `t2.micro`
- AMI: `Amazon Linux 2023`
- Region: `<YOUR_AWS_REGION>`

Security Group rules used:

- Inbound `HTTP` port `80` from `0.0.0.0/0`
- Inbound `SSH` port `22` from `<YOUR_PUBLIC_IP>/32`

The EC2 public IP is:

- `<YOUR_EC2_PUBLIC_IP>`

## 2. Docker Installation on EC2

Commands used on the EC2 host:

```bash
sudo dnf update -y
sudo dnf install -y docker
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ec2-user
```

After adding the user to docker group, reconnect SSH.

## 3. Container Image and Deployment Flow

The pipeline in `.github/workflows/deploy.yml` performs these stages in order on push to `main`:

1. Run tests in `app/` with `npm test`
2. Build Docker image tagged with commit SHA
3. Push image to GitHub Container Registry (GHCR)
4. SSH to EC2, pull image, and restart container

Image format used:

- `ghcr.io/<GHCR_USERNAME>/kora-api:<GIT_SHA>`

Container run command used by pipeline:

```bash
docker run -d \
	--name kora-api \
	--restart unless-stopped \
	-p 80:3000 \
	-e PORT=3000 \
	ghcr.io/<GHCR_USERNAME>/kora-api:<GIT_SHA>
```

## 4. Verifying Container Health

Check container status on EC2:

```bash
docker ps
```

Expected: container `kora-api` is `Up` and mapped to `0.0.0.0:80->3000/tcp`.

Test API health from local machine:

```bash
curl http://<YOUR_EC2_PUBLIC_IP>/health
```

Expected response:

```json
{"status":"ok"}
```

## 5. Viewing Application Logs

View recent logs:

```bash
docker logs --tail 100 kora-api
```

Follow live logs:

```bash
docker logs -f kora-api
```

## 6. Secrets and Access Control

GitHub repository secrets configured:

- `GHCR_USERNAME`
- `GHCR_TOKEN`
- `EC2_HOST`
- `EC2_USER`
- `EC2_SSH_KEY`

No secrets or private keys are stored in source code.

The pipeline uses only required credentials for image push and remote deploy.
