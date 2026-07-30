# BMS — Book My Show

A movie and event ticket booking React SPA — BookMyShow clone.

## Deployment Pipeline

```mermaid
flowchart LR
    A[Push to main] --> B[GitHub Actions]
    B --> C[SSH into EC2]
    C --> D[git pull]
    D --> E[docker compose up -d --build]
    E --> F[App served on port 80]
```

Every push to the `main` branch triggers automatic deployment to an AWS EC2 instance.

---

## Prerequisites

- An **AWS EC2** instance (Ubuntu) with:
  - Docker and Docker Compose installed
  - Git installed
  - Port **80** open in the security group
- **GitHub Secrets** configured in the repository:
  - `EC2_HOST` — public IP or DNS of the EC2 instance
  - `EC2_USER` — SSH username (e.g., `ubuntu`, `ec2-user`)
  - `EC2_SSH_KEY` — the full private key (PEM) used to SSH into the instance
- **This repository cloned on the instance** at `/home/<user>/bms-github-actions` before the first workflow run (the `git pull` step will fail if the directory doesn't exist)

---

## GitHub Actions Workflow

File: `.github/workflows/deploy.yml`

```yaml
name: Deploy Book My Show App on AWS EC2 Instance

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v6

      - name: SSH into EC2 and deploy
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd /home/${{ secrets.EC2_USER }}/bms-github-actions
            git pull
            docker compose up -d --build
```

### What happens on push

| Step | Action                                                                        |
| ---- | ----------------------------------------------------------------------------- |
| 1    | GitHub Actions runner checks out the latest code                              |
| 2    | Uses `appleboy/ssh-action` to connect to EC2 via SSH                          |
| 3    | On EC2: pulls the latest code from `main`                                     |
| 4    | Rebuilds and restarts the Docker container via `docker compose up -d --build` |

---

## Docker Setup

### Multi-stage Dockerfile (`Dockerfile`)

| Stage     | Base Image            | Purpose                                                          |
| --------- | --------------------- | ---------------------------------------------------------------- |
| `builder` | `node:22-alpine`      | Installs dependencies, runs `vite build` → outputs `dist/`       |
| (final)   | `nginx:stable-alpine` | Serves the built `dist/` from `/usr/share/nginx/html` on port 80 |

### docker-compose.yml

```yaml
services:
  app:
    build: .
    restart: unless-stopped
    ports:
      - 80:80
```

Maps host port **80** to container port **80**. The container automatically restarts unless explicitly stopped.

---

## EC2 Instance Setup

These steps are required **once** on the EC2 instance.

### 1. Install Docker & Docker Compose

```bash
# Install Docker
sudo apt update && sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER

# Log out and back in, then install Compose plugin
sudo apt install -y docker-compose-plugin
```

### 2. Clone the repository

```bash
git clone <repo-url> /home/<user>/bms-github-actions
```

> The directory name must match the path in the GitHub Actions script.

### 3. Open port 80

In the EC2 security group, add an inbound rule:

| Type | Protocol | Port | Source    |
| ---- | -------- | ---- | --------- |
| HTTP | TCP      | 80   | 0.0.0.0/0 |

---

## Setting Up GitHub Secrets

In your GitHub repository, go to **Settings → Secrets and variables → Actions** and add:

| Secret        | Value                                                                                                                            |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `EC2_HOST`    | Public IPv4 address or DNS name of EC2                                                                                           |
| `EC2_USER`    | SSH username (`ubuntu` for Ubuntu AMI)                                                                                           |
| `EC2_SSK_KEY` | The complete private key file contents (including `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----`) |

---

## Screenshots

### GitHub Actions Workflow Run

![GitHub Actions workflow](assets/images/github-action.png)

The automated deployment pipeline triggered on a push to `main` — checking out code, SSH-ing into EC2, pulling latest changes, and rebuilding the Docker container.

### EC2 with Docker Container Running

![EC2 Docker container](assets/images/ec2-docker-container.png)

The EC2 instance serving the app via Docker. The container runs the nginx image built from the multi-stage Dockerfile, exposed on port 80.

### Application UI

![BMS App](assets/images/app-image.png)

The Book My Show frontend — a React SPA for browsing and booking movie and event tickets.

---

## Local Development

```bash
npm install          # Install dependencies
npm start            # Dev server at localhost:5173
npm run build        # Production build to dist/
npm run preview      # Preview production build
npm run lint         # Run ESLint
```

---
