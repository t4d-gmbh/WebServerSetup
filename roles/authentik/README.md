# Ansible Role: Authentik

[![build](https://img.shields.io/github/actions/workflow/status/t4d-gmbh/WebServerSetup/molecule-authentik.yml?label=build)](https://github.com/t4d-gmbh/WebServerSetup/actions/workflows/molecule-authentik.yml)

This Ansible role installs and configures [Authentik](https://goauthentik.io/), an open-source Identity Provider, as a Docker Compose stack. It deploys the Authentik server, worker, PostgreSQL database, an automated database backup service, and optionally Redis (for versions prior to 2025.10.0). The role integrates with Traefik for reverse proxying and TLS termination.

## Requirements

- Ansible 2.9 or higher
- Access to a server with `apt` package manager
- Docker and Docker Compose installed
- A user with permissions to manage Docker containers
- Traefik configured as a reverse proxy (the Authentik server container uses Traefik labels for routing)
- The `community.docker` Ansible collection

## Role Variables

### Variables with Defaults

| Variable | Default | Description |
|---|---|---|
| `docker_user` | `'dancer'` | The system user that owns Authentik files and runs the Docker Compose stack. |
| `authentik_base_path` | `'/data/docker/authentik'` | Base directory for all Authentik configuration, data, and Docker Compose files. |
| `AUTHENTIK_CONTAINER_NAME` | `'authentik-server'` | Container name for the Authentik server service. |
| `AUTHENTIK_TAG` | `"2025.6.3"` | Authentik Docker image tag. Also controls whether Redis is included (skipped for >= 2025.10.0). |
| `AUTHENTIK.server_url` | `"https://auth.myserver.net"` | The URL at which Authentik will be accessible. Used in Traefik routing labels. |

### Required Variables (no default)

| Variable | Description |
|---|---|
| `AUTHENTIK_ENV` | A dictionary of key-value pairs written to the `.env` file. Must contain at least `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`, `AUTHENTIK_SECRET_KEY`, and `AUTHENTIK_TAG`. |
| `dns_provider` | The DNS provider name used as the Traefik TLS cert resolver (e.g., `infomaniak`, `cloudflare`). |

### Required Keys in `AUTHENTIK_ENV`

The `.env` file is consumed by all services in the Docker Compose stack. The following keys must be present in the `AUTHENTIK_ENV` dictionary:

| Key | Description |
|---|---|
| `POSTGRES_USER` | PostgreSQL username |
| `POSTGRES_PASSWORD` | PostgreSQL password |
| `POSTGRES_DB` | PostgreSQL database name |
| `AUTHENTIK_SECRET_KEY` | Secret key for Authentik |
| `AUTHENTIK_TAG` | Authentik image tag (should match the `AUTHENTIK_TAG` Ansible variable) |

Optional keys include `AUTHENTIK_IMAGE` (defaults to `ghcr.io/goauthentik/server`) and `AUTHENTIK_ERROR_REPORTING__ENABLED` (defaults to `true`).

## Dependencies

This role requires the Docker role (`t4d.WebServerSetup.docker_setup`) to be executed prior to this role. It also expects a running Traefik instance (e.g., via `t4d.WebServerSetup.traefik`) for reverse proxy integration.

## Tasks Overview

1. **Ensure Docker User Exists**: Ensures the specified system user is present.
2. **Create Docker Networks**: Creates external `proxy` and `backup` Docker networks.
3. **Create Configuration Directory**: Creates the base directory and subdirectories (`certs`, `custom-templates`, `backups`, `database`, `media`, `redis`).
4. **Create Backup Service Files**: Generates a `Dockerfile.backup` and `entrypoint.sh` for a PostgreSQL backup container that runs a daily `pg_dump` at 02:00.
5. **Template Docker Compose File**: Renders the `docker-compose.yml` from the Jinja2 template, conditionally including Redis based on the Authentik version.
6. **Template .env File**: Renders the `.env` file from the `AUTHENTIK_ENV` dictionary.
7. **Install ACL Package**: Installs the `acl` system package (required for Ansible `become_user` with non-root users).
8. **Start Authentik Stack**: Brings up all services via `docker_compose_v2`.

## Services Deployed

| Service | Container Name | Description |
|---|---|---|
| `postgresql` | `authentik-postgresql` | PostgreSQL 16 database (Alpine) |
| `backup` | `authentik-postgresql-backup` | Daily automated `pg_dump` backup (cron at 02:00) |
| `redis` | `authentik-redis` | Redis cache (only for Authentik < 2025.10.0) |
| `server` | configurable via `AUTHENTIK_CONTAINER_NAME` | Authentik web server (port 9000, routed via Traefik) |
| `worker` | `authentik-worker` | Authentik background worker (runs as root, mounts Docker socket) |

## Example Playbook

```yaml
- hosts: webservers
  become: yes
  vars:
    docker_user: "dancer"
    dns_provider: "infomaniak"
    AUTHENTIK_TAG: "2025.6.3"
    AUTHENTIK:
      server_url: "https://auth.example.com"
    # This should be defined in a vault:
    AUTHENTIK_ENV:
      POSTGRES_USER: "authentik"
      POSTGRES_PASSWORD: "{{ vault_authentik_db_password }}"
      POSTGRES_DB: "authentik"
      AUTHENTIK_SECRET_KEY: "{{ vault_authentik_secret_key }}"
      AUTHENTIK_TAG: "2025.6.3"
  roles:
    - t4d.WebServerSetup.docker_setup
    - t4d.WebServerSetup.traefik
    - t4d.WebServerSetup.authentik
```

## Notes

- **Backups**: The backup service writes daily PostgreSQL dumps to `{{ authentik_base_path }}/backups/authentik_postgres_backup.tar`. Each dump overwrites the previous one -- consider adding rotation or off-site backup if needed.
- **Redis**: Redis is automatically included for Authentik versions before 2025.10.0 and omitted for 2025.10.0 and later, as Authentik dropped the Redis dependency at that version.
- **Worker permissions**: The worker container runs as `root` and mounts `/var/run/docker.sock` -- this is required for Authentik's outpost management.
- **Sensitive data**: Store `POSTGRES_PASSWORD` and `AUTHENTIK_SECRET_KEY` in an Ansible vault rather than in plain text.

## License

This role is licensed under the GNU GPLv3 License.

## Author Information

This role was created in 2025 by Jonas I Liechti @ T4D.ch.
