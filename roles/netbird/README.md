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
- `netbird_proxy_enabled`: Deploy the NetBird reverse proxy (`netbirdio/reverse-proxy`) to expose internal services publicly (default: false). See [Reverse proxy feature](#reverse-proxy-feature).
- `netbird_proxy_domain`: Base domain of the proxy cluster; services land on `<name>.<domain>`. Defaults to the management host (official quickstart behaviour); set a dedicated domain (with an A record) to keep it separate from `server_url`.
- `netbird_proxy_name`: Name of the generated proxy access token (default: "netbird-proxy").
- `netbird_proxy_port`: Proxy listener port inside the container (default: 8443; reached via Traefik TLS passthrough on 443).
- `netbird_proxy_wg_port`: UDP port published for the proxy's embedded WireGuard client (default: 51820).
- `netbird_proxy_sni_excludes`: Hostnames Traefik must keep for its own HTTP routers. The passthrough rule is a catch-all (`HostSNI(*)`); `netbird_host` is always excluded automatically — add **every other service hostname** served by the shared Traefik (authentik, headscale, opencpu, ...), or their TLS will be passed to the proxy.
- `netbird_proxy_trusted_proxies`: CIDRs whose PROXY protocol headers the proxy trusts. Auto-detected from the `proxy` docker network when empty.
- `netbird_trusted_http_proxies`: Optional list of trusted reverse-proxy CIDRs for client-IP forwarding (default: `[]`).
- `server_url`: Public URL of the NetBird instance, e.g. `https://netbird.example.com` (from the Traefik/Headscale convention of this collection).
- `dns_provider`: Traefik certificate resolver name used for TLS (e.g. `infomaniak`).

Vault variables (required, define them in your `vault.yml`):

- `vault_netbird_auth_secret`: Shared secret for relay authentication. Generate with `openssl rand -base64 32`.
- `vault_netbird_store_encryption_key`: base64-encoded 32-byte key encrypting setup keys and tokens at rest. Generate with `openssl rand -base64 32`. **Back this key up** — losing it means losing access to encrypted data.
- `vault_netbird_session_cookie_key`: Optional AES key for embedded IdP session cookies. Generate with `openssl rand -base64 32`.
- `vault_netbird_proxy_token`: Optional pre-created proxy access token (`nbx_...`). When unset and `netbird_proxy_enabled` is true, the role generates one via the management CLI and persists it as `.proxy_token` in the base path.

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
4. Create the first admin user (see [First admin user](#first-admin-user)), then log in at `https://netbird.example.com` — with the dashboard URL, create peers, setup keys, and policies. Clients connect with `netbird setup --key <setup-key>`.

## First admin user

The role intentionally does not bootstrap the admin account — matching the official quickstart, the first owner is created through the setup flow, which stores the password correctly (bcrypt via the embedded IdP).

After the first playbook run, open `https://<server_url>/setup` and create the admin account there (password: at least 8 characters including one digit, one uppercase letter, and one special character). The `/setup` page is only available while no user exists; once an account exists it redirects to the login page.

For automated deployments, complete setup via the API instead:

```bash
curl -X POST "https://<server_url>/api/setup" \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@example.com", "password": "NewPass1!", "name": "Admin"}'
```

If a local user locks themselves out, an administrator can reset the password on the server with the admin CLI (no data loss):

```bash
printf '%s\n' 'NewPass1!' | docker exec -i netbird-server \
  /go/bin/netbird-server --config /etc/netbird/config.yaml admin user change-password \
  --email admin@example.com --password-file -
```

Using `--password-file -` reads the password from stdin so it never appears in the process arguments.

## Reverse proxy feature

Setting `netbird_proxy_enabled: true` adds a third container (`netbird-proxy`, image `netbirdio/reverse-proxy`) that exposes internal services to the public internet. The configuration mirrors NetBird's official quickstart:

- The proxy manages its **own TLS certificates** (Let's Encrypt, `tls-alpn-01`), so Traefik forwards matching connections as **raw TLS passthrough** — no termination.
- The passthrough router is a catch-all (`HostSNI(*)`, lowest priority) minus explicit exclusions: `netbird_host` is always excluded, and **every other hostname on the shared Traefik must be added to `netbird_proxy_sni_excludes`** — otherwise its TLS is handed to the proxy.
- The proxy connects to management over the docker network (`http://netbird-server:80`), so no `/management.ProxyService/` route through Traefik is needed.
- The traefik role's `websecure` entrypoint needs `allowACMEByPass: true` (set in the `traefik-3.7.13.yaml.j2` template) so the proxy can solve its own TLS-ALPN challenges.

**Enabling steps:**

1. DNS: an A record for `netbird_proxy_domain` pointing at the server, plus a wildcard CNAME `*.<netbird_proxy_domain> → <netbird_proxy_domain>` if you want services on the cluster domain itself.
2. Playbook: `netbird_proxy_enabled: true` (and `netbird_proxy_sni_excludes` with your other service hostnames), then run.
3. The role generates the proxy access token once (`admin token create --name <netbird_proxy_name>`) and persists it in `.proxy_token` — the token is **never rotated** on later runs; revoke/recreate via `admin token list|revoke` and delete `.proxy_token` to force regeneration.
4. Verify: dashboard → **Reverse Proxy → Services** — the domain appears with a *Cluster* badge.

**Custom domains:** added purely in the dashboard (*Reverse Proxy → Custom Domains*), verified via a wildcard CNAME pointing at `netbird_proxy_domain`. They need **no role or Traefik change** — the catch-all passes their TLS through automatically (check CAA records allow `letsencrypt.org` if your zone publishes any).

**Requirements & caveats:**

- Port 443 must be reachable (passthrough) and UDP 51820 published for the proxy's WireGuard client.
- The proxy's embedded client dials the **public URL** of the management server; if your network does not hairpin the server's own public IP, proxied services return 504 (`failed connecting to Signal Service`). Workaround: split DNS for the management domain inside the proxy container (`extra_hosts`).
- TCP/UDP (L4) proxy services need per-port mappings on the proxy container — not managed by this role; add them via your own compose overrides.
- CrowdSec integration is not managed by this role.

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
