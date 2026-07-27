# DevOps Assignment — Real-Time Chat App

A real-time chat application deployed with Docker, served through Nginx, and shipped to AWS EC2 via a GitHub Actions CI/CD pipeline.

**Live demo:** http://13.205.60.76

---

## Table of contents

- [Project overview](#project-overview)
- [Assignment objective](#assignment-objective)
- [Architecture](#architecture)
- [Tech stack](#tech-stack)
- [Project structure](#project-structure)
- [Docker setup](#docker-setup)
- [Networking](#networking)
- [Issues identified](#issues-identified)
- [Fixes implemented](#fixes-implemented)
- [Deployment steps](#deployment-steps)
- [CI/CD with GitHub Actions](#cicd-with-github-actions)
- [Screenshots](#screenshots)
- [Future improvements](#future-improvements)

---

## Project overview

This project takes a broken real-time chat application (FastAPI backend + static frontend) and turns it into a properly containerized, reverse-proxied, and automatically deployed service running on AWS EC2.

## Assignment objective

Given a starter chat application with intentional configuration gaps, the goals were to:

1. Fix the Docker and Docker Compose setup so both services build and run correctly.
2. Configure Nginx to serve the static frontend and reverse-proxy WebSocket traffic to the backend.
3. Deploy the stack to a cloud VM (AWS EC2).
4. Automate deployment with a CI/CD pipeline (GitHub Actions).

## Architecture

```
                Git Push
                    │
                    ▼
           GitHub Repository
                    │
                    ▼
          GitHub Actions (CI/CD)
                    │ SSH deploy
                    ▼
             AWS EC2 Instance
                    │
            Docker Compose
          ┌─────────┴─────────┐
          │                   │
          ▼                   ▼
     Nginx Container    FastAPI Container
     (port 80, public)   (port 8000, internal)
          │                   │
          └─────────┬─────────┘
                     │ proxies /ws
                     ▼
                  Browser
```

Nginx is the only container with a published port (`80:80`). It serves the static frontend directly from disk and reverse-proxies `/ws` WebSocket traffic to the FastAPI backend container over the internal Docker network. The backend container has no published port — it's reachable only from other containers on `devops-assignment_default`.

## Tech stack

| Layer | Technology |
|---|---|
| Backend | FastAPI + Uvicorn (Python 3.11) |
| Frontend | Static HTML/JS |
| Reverse proxy / web server | Nginx (alpine) |
| Containerization | Docker, Docker Compose |
| CI/CD | GitHub Actions |
| Hosting | AWS EC2 (Ubuntu 24.04) |

## Project structure

```
devops-assignment/
├── .github/
│   └── workflows/          # GitHub Actions deployment workflow
├── app/
│   ├── main.py              # FastAPI application
│   └── requirements.txt
├── frontend/                 # Static chat UI served by Nginx
├── Dockerfile                # Backend image build
├── docker-compose.yml        # Backend + Nginx services
├── nginx.conf                 # Reverse proxy + static file config
├── .dockerignore
├── .gitignore
└── README.md
```

## Docker setup

Two services are defined in `docker-compose.yml`:

- **backend** — built from the local `Dockerfile`, runs `uvicorn main:app`, exposes port `8000` internally only (`expose`, not `ports`).
- **nginx** — official `nginx:alpine` image, publishes port `80`, mounts the `frontend/` directory read-only for static files and `nginx.conf` for the reverse-proxy configuration.

Both services restart automatically (`restart: always`) and share the default Compose bridge network, so the Nginx container can reach the backend by its service name (`chat-backend`) without any manual network configuration.

## Networking

- Docker Compose creates an isolated bridge network (`devops-assignment_default`) so the two containers can resolve each other by container name.
- Nginx listens on `0.0.0.0:80`, mapped to the EC2 instance's public interface.
- WebSocket upgrade headers (`Upgrade`, `Connection: Upgrade`) are explicitly set in the `/ws` location block so the proxy doesn't fall back to plain HTTP.
- The AWS security group for the EC2 instance allows inbound traffic on port 80 (HTTP) and 22 (SSH).

## Issues identified

The starter project had several gaps that prevented it from running out of the box:

- The frontend directory wasn't correctly mounted into the Nginx container.
- Nginx wasn't configured to proxy WebSocket upgrade headers, so real-time messages failed silently.
- The Compose file used an obsolete `version` attribute, producing warnings on modern Docker versions.
- There was no CI/CD pipeline — deployment was entirely manual.

## Fixes implemented

- Corrected the `frontend:/usr/share/nginx/html` volume mount in `docker-compose.yml`.
- Added a dedicated `/ws` location block in `nginx.conf` with proper `Upgrade`/`Connection` headers and `proxy_http_version 1.1`.
- Removed the obsolete `version` field from `docker-compose.yml`.
- Added a `.dockerignore` to keep build context small.
- Wrote a GitHub Actions workflow that SSHes into the EC2 instance and redeploys on every push to `main`.

## Deployment steps

1. **Provision the server** — launch an AWS EC2 instance (Ubuntu 24.04), attach an Elastic IP, and open ports 22 and 80 in the security group.
2. **Install Docker** on the instance (`docker` + `docker compose` plugin).
3. **Clone the repository** into `/var/www/devops-assignment`.
4. **Build and run**:
   ```bash
   docker compose up -d --build
   ```
5. **Verify**:
   ```bash
   docker compose ps
   docker logs chat-backend
   docker logs chat-nginx
   ```
6. **Confirm public access** at `http://<elastic-ip>`.

## CI/CD with GitHub Actions

A workflow in `.github/workflows/` triggers on every push to `main`:

1. Connects to the EC2 instance over SSH using a repository secret.
2. Pulls the latest code.
3. Runs `docker compose down` followed by `docker compose up -d --build` to rebuild and restart both containers.

This means any change pushed to `main` is live on the server within a couple of minutes, with no manual steps.

## Screenshots

Screenshots live in the `screenshots/` folder and cover:

- Repository structure
- `docker compose up --build` output
- Successful GitHub Actions run
- `docker ps` / `docker compose ps` showing both containers healthy
- The live application in the browser, with chat working end-to-end

*(Add the images to `screenshots/` and reference them here, e.g. `![Containers running](screenshots/docker-ps.png)`.)*

## Future improvements

- Add HTTPS via Let's Encrypt / Certbot once a stable domain is attached.
- Add a health-check endpoint and Docker `HEALTHCHECK` directives for both containers.
- Persist chat history in a database instead of in-memory state.
- Add automated tests to the GitHub Actions workflow before deployment.
