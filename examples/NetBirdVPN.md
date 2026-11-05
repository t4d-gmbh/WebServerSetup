<!--
SPDX-FileCopyrightText: 2026 Jonas I. Liechti <j-i-l@t4d.ch>

SPDX-License-Identifier: GPL-3.0-or-later
-->

# NetBird VPN Setup 🕸️

This README provides instructions on how to use three Ansible roles: **Docker**, **Traefik**, and **NetBird**. These roles are designed to set up a Docker environment with Traefik as a reverse proxy and a self-hosted [NetBird](https://netbird.io) zero-trust VPN platform (combined management, signal, relay, and STUN server plus the web dashboard).

## Prerequisites

- Ansible installed on your control machine.
- Access to a target machine (e.g., an Ubuntu server with at least 1 CPU and 2 GB memory) where the roles will be applied.
- SSH access to the target machine using an SSH key.
- A public domain name (e.g. `netbird.myserver.net`) with an A record pointing to the server.
- The server publicly reachable on **TCP 80/443** (Traefik) and **UDP 3478** (NetBird STUN).

## Directory Structure

Ensure your project directory has the following structure:

```
project/
├── playbook.yml
├── vault.yml
├── requirements.yml
└── inventory.ini
```

## Installation

You will need the [T4D.WebServerSetup](https://github.com/t4d/WebServerSetup) Ansible collection.

The simplest way to add it is by creating a `requirements.yml` file in your project with the following content:

```yaml
---
collections:
  - name: t4d.WebServerSetup
    type: git
    source: https://github.com/t4d-gmbh/WebServerSetup.git
    version: main
```

To install the requirements, simply run:

```
ansible-galaxy install -r requirements.yml
```

## Usage

In short, you need:

- A playbook that installs the roles **docker**, **traefik**, and **netbird** on your target machine.
- An `inventory.ini` file that specifies the target machine and how to access it.
- A `vault.yml` file that contains all the necessary parameters.

To run the playbook and deploy your NetBird server, use the following command:

```
ansible-playbook playbook.yml -i inventory.ini --ask-vault-pass
```

### Playbook Example

Here’s an example of the playbook file (`playbook.yml`):

```yaml
---
- name: Deploy NetBird behind Traefik
  hosts: all
  become: true
  vars_files:
    - vault.yml  # Load variables from the vault
  vars:
    dancer_user: "dancer"
    server_url: "https://netbird.myserver.net"
    email: "admin@example.com"  # Email for ACME
    dns_provider: "infomaniak"  # DNS provider

  tasks:
    - name: Ensure Docker is set up and running
      include_role:
        name: t4d.WebServerSetup.docker

    - name: Create Docker network 'proxy'
      community.docker.docker_network:
        name: proxy
        state: present

    - name: Ensure Traefik is configured
      include_role:
        name: t4d.WebServerSetup.traefik

    - name: Ensure NetBird is configured
      include_role:
        name: t4d.WebServerSetup.netbird
```

### Inventory File

An example of an inventory file (`inventory.ini`) is as follows:

```
[all]
netbird.myserver.net ansible_ssh_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

The following values are examples and need to be adapted to your specific settings:

- `netbird.myserver.net`: The server URL to deploy the roles to.
- `ubuntu`: The user that can access the server.
- `~/.ssh/id_ed25519`: Path to the SSH private key to access the server.

### Vault File

To create a `vault.yml` file, run the command:

```
ansible-vault create vault.yml
```

Then copy and paste the following content, adapting it as necessary:

```yaml
CERTRESOLVER:
  INFOMANIAK_ACCESS_TOKEN: "sNphaL-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
  INFOMANIAK_ENDPOINT: "https://api.infomaniak.com"
vault_netbird_auth_secret: "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX="
vault_netbird_store_encryption_key: "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX="
```

Under `CERTRESOLVER`, you can place all the necessary variables so that Traefik can request and renew the certificate from your specific provider. A list of providers and required variables can be found in [Traefik's official documentation](https://doc.traefik.io/traefik/https/acme/#providers).

Generate the two NetBird secrets with:

```
openssl rand -base64 32
```

Keep `vault_netbird_store_encryption_key` safe and backed up: it encrypts setup keys and API tokens in the database, and losing it means losing access to that data.

If you need to further edit the existing `vault.yml`, use:

```
ansible-vault edit vault.yml
```

## First Login

Once the playbook has run, open `https://netbird.myserver.net/setup` and create the first admin account there (the setup page is only available while no user exists; the password needs at least 8 characters including one digit, one uppercase letter, and one special character). For automated deployments the same can be done via `POST /api/setup` — see the [role README](../roles/netbird/README.md#first-admin-user).

You can then create setup keys in the dashboard and enroll clients with `netbird up --setup-key <key>`.

## Reverse Proxy (optional)

NetBird can also expose internal services to the public internet through a dedicated proxy container. To enable it, add the proxy variables to the playbook `vars` and rerun:

```yaml
    netbird_proxy_enabled: true
    netbird_proxy_domain: "proxy.myserver.net"   # or omit to reuse the NetBird host
    netbird_proxy_sni_excludes:                   # every other hostname on this Traefik!
      - "auth.myserver.net"
```

Then:

1. Create DNS records: A `proxy.myserver.net` → server IP, and (to host services on the cluster domain) CNAME `*.proxy.myserver.net` → `proxy.myserver.net`.
2. Rerun the playbook — the role generates the proxy access token once and starts `netbird-proxy` behind Traefik TLS passthrough.
3. Verify in the dashboard under **Reverse Proxy → Services**: the domain shows with a *Cluster* badge.

Custom domains for services are added entirely in the dashboard (*Reverse Proxy → Custom Domains*, verified via a wildcard CNAME pointing at the proxy domain) — no playbook change needed. See the [role README](../roles/netbird/README.md#reverse-proxy-feature) for details and caveats (UDP 51820, hairpin NAT, token lifecycle).

## Used Roles

Here can find detailed information about the roles used in this examples:

- [Docker](../roles/docker/README.md)
- [Traefik](../roles/traefik/README.md)
- [NetBird](../roles/netbird/README.md)
