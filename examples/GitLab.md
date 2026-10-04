<!--
SPDX-FileCopyrightText: 2026 Jonas I. Liechti <j-i-l@t4d.ch>

SPDX-License-Identifier: GPL-3.0-or-later
-->

# GitLab CE behind NetBird (VPN-only server) 🦊

This example deploys a self-hosted **GitLab CE** server that is reachable **only from inside a NetBird network** — there is no public HTTP/HTTPS port. Because Let's Encrypt can never reach the machine, TLS certificates are obtained with **certbot's DNS-01 challenge** (Infomaniak), and the host joins the VPN with the **netbird client** role.

## Architecture

```
 laptop (netbird client) ──▶ 100.64.x.y (netbird overlay) ──▶ GitLab VM (Ubuntu 24.04)
                                     ▲
 bird.t42d.ch (public) ──────────────┘  management plane only
```

- The **netbird server** (`t4d.WebServerSetup.netbird` + traefik + docker) runs on the public host — see [NetBirdVPN.md](NetBirdVPN.md).
- The **GitLab VM** runs Ubuntu 22.04/24.04 (omnibus repos exist only for LTS), installs the netbird client, gets a `100.64.x.y` address, and serves HTTPS itself.
- DNS: an `A` record `gitlab.myserver.net → <netbird IP>` (the overlay address). It does not have to be reachable from the public internet — DNS-01 validates through the Infomaniak API, not through the host.

## Prerequisites

- A running NetBird deployment with a **setup key** (dashboard → Peers → Access Keys).
- A second Ubuntu 24.04 VM reachable once via normal SSH (it may live behind the VPN later; SSH should stay available to you).
- Infomaniak DNS API token (same one the traefik role uses).
- ≥ 4 GB RAM (8 GB recommended) on the GitLab VM.

## Directory Structure

```
project/
├── playbook.yml
├── vault.yml
├── requirements.yml
└── inventory.ini
```

`requirements.yml` as in [NetBirdVPN.md](NetBirdVPN.md).

## Playbook

```yaml
---
- name: GitLab VPN enrollment
  hosts: gitlab
  become: true
  vars_files:
    - vault.yml
  vars:
    netbird_client_management_url: "https://bird.t42d.ch"
  roles:
    - t4d.WebServerSetup.netbird_client

- name: GitLab CE with DNS-01 TLS
  hosts: gitlab
  become: true
  vars_files:
    - vault.yml
  vars:
    server_url: "https://gitlab.myserver.net"
    domain_name: "gitlab.myserver.net"          # consumed by certbot
    certbot_email: "admin@example.com"
    cert_path: "/etc/letsencrypt/live/gitlab.myserver.net/fullchain.pem"
    key_path: "/etc/letsencrypt/live/gitlab.myserver.net/privkey.pem"
    certbot_renew_hook: "gitlab-ctl restart nginx"
  roles:
    - t4d.WebServerSetup.certbot   # issues the DNS-01 certificate first
    - t4d.WebServerSetup.gitlab    # omnibus uses those certificate paths
```

Order matters: **certbot before gitlab** — the gitlab role asserts that the certificate files exist before reconfigure, because omnibus nginx will not start without them.

## Inventory

```
[all]
gitlab-vm ansible_host=<SSH-reachable address> ansible_ssh_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

## Vault

```yaml
CERTRESOLVER:
  INFOMANIAK_ACCESS_TOKEN: "sNphaL-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
  INFOMANIAK_ENDPOINT: "https://api.infomaniak.com"
vault_infomaniak_DNS_key: "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"   # certbot DNS-01
vault_netbird_setup_key: "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
vault_gitlab_initial_root_password: "ChooseARobustRootPassword123!"
```

## Run

```
ansible-playbook playbook.yml -i inventory.ini --ask-vault-pass
```

The first `gitlab-ctl reconfigure` takes 10–20 minutes. Subsequent runs are fast; reconfigure only triggers when `gitlab.rb` or the package version change.

## First Login

- With `vault_gitlab_initial_root_password` set: log in as `root` with that password at `https://gitlab.myserver.net`.
- Without it: read `/etc/gitlab/initial_root_password` on the host (valid for 24 hours only).

Then create projects, users and (optionally) enable the container registry in **Admin Area**.

## Accessing GitLab from your machines

1. Enroll your laptop in the same NetBird network (`netbird up --setup-key ... --management-url https://bird.t42d.ch` or the desktop app).
2. `git clone https://gitlab.myserver.net/<group>/<project>.git` — the A record resolves to the overlay IP and the connection goes through the VPN.

## Certificate renewal

The certbot role's cron renews daily and runs `gitlab-ctl restart nginx` via `certbot_renew_hook` only after an actual renewal. No GitLab data is touched.

## Caveats

- **Git LFS / large uploads**: omnibus nginx defaults are generous; if you put anything in front later, revisit proxy body limits.
- **Container registry**: not enabled by this role; add `registry_external_url` wiring to `gitlab.rb.j2` via a `gitlab_rails`/`registry` extension if needed.
- **Upgrades**: bump `gitlab.version` stepwise per GitLab's upgrade-path notes (never skip minors across major boundaries) — the handler reconfigures automatically after the package changes.

## Used Roles

- [NetBird Client](../roles/netbird_client/README.md)
- [Certbot](../roles/certbot/README.md)
- [GitLab](../roles/gitlab/README.md)
