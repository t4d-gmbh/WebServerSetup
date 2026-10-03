# Ansible Role: NetBird

[![build](https://img.shields.io/github/actions/workflow/status/t4d-gmbh/WebServerSetup/molecule-netbird.yml?label=build)](https://github.com/t4d-gmbh/WebServerSetup/actions/workflows/molecule-netbird.yml)

This Ansible role installs and configures a self-hosted [NetBird](https://netbird.io) platform. It deploys the combined NetBird server (management, signal, relay, and embedded STUN in one container) and the web dashboard as Docker containers, integrated with the collection's Traefik reverse proxy for TLS termination and routing. User authentication uses NetBird's embedded identity provider (Dex).

## Requirements

- Ansible 2.9 or higher
- Docker and Docker Compose installed (use the `docker` role from this collection)
- Traefik running on the external `proxy` network (use the `traefik` role from this collection)
- A public domain name (e.g. `netbird.example.com`) resolving to the server
- The server must be publicly reachable on **TCP 80/443** (via Traefik) and **UDP 3478** (STUN, exposed directly — it cannot be proxied)
- At least 1 CPU and 2 GB of memory

## Role Variables

- `dancer_user`: The username that owns the NetBird files and runs the containers (default: "dancer").
- `netbird_base_path`: Directory for NetBird configuration (default: `/data/docker/netbird`).
- `netbird.server_tag`: Image tag for `netbirdio/netbird-server` (default: "0.80.0").
- `netbird.dashboard_tag`: Image tag for `netbirdio/dashboard` (default: "v2.94.0").
- `netbird_stun_port`: UDP port of the embedded STUN server (default: 3478).
- `netbird_log_level`: Log level of the NetBird server (default: "info").
- `netbird_store_engine`: Management store engine, `sqlite` or `postgres` (default: "sqlite").
- `netbird_admin_email`: Optional email for the initial admin user. When set, an `owner` block is rendered into `config.yaml` and the account is created on first startup. When empty, create the first user manually via the `/setup` page.
- `netbird_trusted_http_proxies`: Optional list of trusted reverse-proxy CIDRs for client-IP forwarding (default: `[]`).
- `server_url`: Public URL of the NetBird instance, e.g. `https://netbird.example.com` (from the Traefik/Headscale convention of this collection).
- `dns_provider`: Traefik certificate resolver name used for TLS (e.g. `infomaniak`).

Vault variables (required, define them in your `vault.yml`):

- `vault_netbird_auth_secret`: Shared secret for relay authentication. Generate with `openssl rand -base64 32`.
- `vault_netbird_store_encryption_key`: base64-encoded 32-byte key encrypting setup keys and tokens at rest. Generate with `openssl rand -base64 32`. **Back this key up** — losing it means losing access to encrypted data.
- `vault_netbird_admin_password`: Password for `netbird_admin_email`.
- `vault_netbird_session_cookie_key`: Optional AES key for embedded IdP session cookies. Generate with `openssl rand -base64 32`.

## Dependencies

This role requires the `docker` role to be executed prior to this role, and expects the Traefik container of the `traefik` role to serve the `websecure` entrypoint on the shared `proxy` network.

## Installation

To use this role, add it to your Ansible playbook as follows:

```yaml
- hosts: your_target_hosts
  roles:
    - t4d.WebServerSetup.netbird
```

## Tasks Overview

1. **Ensure User Exists**: Checks that the Docker user exists on the system.
2. **Create Proxy Docker Network**: Ensures the external `proxy` network exists.
3. **Create NetBird Configuration Directory**: Creates `/data/docker/netbird`.
4. **Create NetBird Server Configuration File**: Renders `config.yaml` from the template (mode 0600, contains secrets).
5. **Create NetBird Dashboard Environment File**: Renders `dashboard.env` (OIDC/endpoint configuration for the dashboard).
6. **Create NetBird Docker Compose File**: Renders `docker-compose.yml` with Traefik labels for gRPC (h2c) and HTTP routing.
7. **Ensure ACL Package is Installed**: Installs the ACL package for `become_user` support.
8. **Start NetBird Containers**: Uses `docker_compose_v2` to start `netbird-server` and `dashboard`.

## Usage

1. Point your domain (A record) at the server.
2. Define the required variables in your playbook or inventory, and the vault variables in `vault.yml`.
3. Run the playbook.
4. Log in at `https://netbird.example.com` — with the dashboard URL, create peers, setup keys, and policies. Clients connect with `netbird setup --key <setup-key>`.

## Example Playbook

```yaml
# netbird.yml
---
- name: Configure NetBird VPN platform
  hosts: all
  vars_files:
    - vault.yml
  vars:
    server_url: https://netbird.example.com
    dns_provider: infomaniak
    email: some@e.mail
    netbird_admin_email: admin@example.com
  roles:
    - t4d.WebServerSetup.docker
    - t4d.WebServerSetup.traefik
    - t4d.WebServerSetup.netbird
```

Run it with:

```bash
ansible-playbook netbird.yml --ask-vault-pass -i your_inventory_file
```

## Notes

- The NetBird management API is reachable at `https://<server_url>/api`. For declarative tenant management (users, groups, setup keys, policies) see the upstream [`community.ansible_netbird`](https://github.com/netbirdio/ansible-netbird) collection — it is not part of this role.
- The embedded Dex identity provider also exposes OIDC at `https://<server_url>/oauth2`, so NetBird can serve as an IdP for other services.
- Backup: the `netbird_data` Docker volume holds the SQLite databases (`store.db`, `events.db`, `idp.db`) and generated keys.
