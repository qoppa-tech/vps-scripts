# Parlor VPS Deployment

![Ansible](https://img.shields.io/badge/Ansible-Playbook-EE0000?logo=ansible&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![Traefik](https://img.shields.io/badge/Traefik-v3.0-24A1C1?logo=traefikproxy&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-18-4169E1?logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-7.4-DC382D?logo=redis&logoColor=white)
[![Deploy](https://github.com/qoppa-tech/vps-scripts/actions/workflows/deploy.yml/badge.svg)](https://github.com/qoppa-tech/vps-scripts/actions/workflows/deploy.yml)

Ansible playbook for provisioning and deploying the Parlor platform to a single VPS. Handles server hardening, Docker setup, reverse proxy, databases, and application deployment via containerized services.

## Architecture

```
Internet
  │
  ▼
Traefik (ports 80/443, TLS via Let's Encrypt)
  ├── landing.parlorharpia.tech → Landing Page (:80)
  └── api.parlorharpia.tech     → API Core (:8080)

Internal (parlor-network)
  ├── PostgreSQL (:5432, localhost only)
  └── Redis (:6379, localhost only)
```

## Roles

| Role | Description |
|---|---|
| `configure_server` | Creates deploy user (`parloruser`), configures SSH keys, disables root login |
| `docker_install` | Installs Docker CE from official repos, logs into GHCR |
| `traefik` | Reverse proxy with automatic HTTPS (Let's Encrypt), HTTP→HTTPS redirect |
| `postgresql` | PostgreSQL 18 container with persistent data volume |
| `redis` | Redis 7.4.3 container with AOF persistence |
| `deploy_apicore` | Go API backend — pulls image from GHCR, runs migrations, deploys |
| `deploy_landing` | Landing page — pulls image from GHCR and deploys |
| `deploy_frontend` | Placeholder for future frontend deployment |

Roles execute in order: server config → Docker → Traefik → PostgreSQL → Redis → API Core → Landing.

## Prerequisites

- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) installed locally
- SSH access to the target VPS via key-based auth
- Secrets file at `group_vars/secrets.yml` (see below)

## Secrets

Create `group_vars/secrets.yml` (gitignored) with:

```yaml
user_password: "<hashed password>"
ssh_authorized_keys:
  - "<your public SSH key>"
postgres_password: "<password>"
redis_password: "<password>"
jwt_secret: "<secret>"
ghcr_user: "<github username>"
ghcr_token: "<github PAT with packages:read>"
mailtrap_token: "<token>"
google_client_id: "<oauth client id>"
google_client_secret: "<oauth client secret>"
```

Or encrypt with Ansible Vault:

```bash
ansible-vault create group_vars/secrets.yml
```

## Usage

Dry run (check mode):

```bash
ansible-playbook -i inventory playbook.yml --check --diff
```

Deploy:

```bash
ansible-playbook -i inventory playbook.yml
```

Deploy a specific role:

```bash
ansible-playbook -i inventory playbook.yml --tags traefik
```

## CI/CD

The GitHub Actions workflow (`.github/workflows/deploy.yml`) runs automatically on pushes to `main` that modify deployment files. It:

1. Runs the playbook in check mode
2. Applies changes only if the dry run detected differences

Secrets are provided via GitHub Actions secrets. Manual runs are also supported via `workflow_dispatch`.
