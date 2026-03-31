# Full Infrastructure Stack

This example deploys a complete container infrastructure on an Ubuntu server: system tuning, Docker engine, Traefik reverse proxy with automatic TLS, Authentik identity provider, and OpenCPU analytics server -- all connected through a shared proxy network.

## Prerequisites

- Ansible installed on your control machine.
- Access to a target machine (e.g., an Ubuntu server) where the roles will be applied.
- SSH access to the target machine using an SSH key (see the [Initial Setup](../README.md#initial-setup) example).
- A domain name with DNS pointing to your server (e.g., `myserver.com`).
- A DNS provider API token for automatic TLS certificate issuance (the example uses Infomaniak).

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

You will need the [T4D.WebServerSetup](https://github.com/t4d-gmbh/WebServerSetup) Ansible collection.

Create a `requirements.yml` file in your project:

```yaml
---
collections:
  - name: t4d.WebServerSetup
    type: git
    source: https://github.com/t4d-gmbh/WebServerSetup.git
    version: main
```

Install the collection:

```
ansible-galaxy install -r requirements.yml
```

## Usage

You need:

- A playbook that installs the roles on your target machine.
- An `inventory.ini` file that specifies the target machine and how to access it.
- A `vault.yml` file that contains all sensitive parameters.

To run the playbook:

```
ansible-playbook playbook.yml -i inventory.ini --ask-vault-pass
```

### Playbook Example

```yaml
# playbook.yml
---
- name: Deploy Full Infrastructure Stack
  hosts: all
  become: true
  vars_files:
    - vault.yml
  vars:
    # System user that will own Docker resources
    dancer_user: "dancer"
    docker_user: "dancer"
    dns_provider: "infomaniak"
    email: "admin@myserver.com"

    # basic_config - swap and kernel tuning
    swapfile_size: "2G"

    # traefik - reverse proxy
    traefik:
      version: "3.5.4"

    # authentik - identity provider
    AUTHENTIK_TAG: "2025.6.3"
    AUTHENTIK:
      server_url: "https://auth.myserver.com"

    # opencpu - R analytics server
    opencpu:
      DOMAIN: "r.myserver.com"
      AUTHENTIK_PROXY_CONTAINER: "authentik-server"
      CONTAINER_NAME: "opencpu"
      DOCKER_IMAGE: "docker.io/opencpu/ubuntu-24.04"
      DOCKER_TAG: "v2.2.14-2"
      RUN_R:
        - "install.packages('pak', repos=c(CRAN='https://cran.r-project.org'))"

  tasks:
    - name: Apply basic server configuration (swap, kernel tuning)
      ansible.builtin.include_role:
        name: t4d.WebServerSetup.basic_config

    - name: Install and configure Docker
      ansible.builtin.include_role:
        name: t4d.WebServerSetup.docker

    - name: Create Docker proxy network
      community.docker.docker_network:
        name: proxy
        state: present

    - name: Deploy Traefik reverse proxy
      ansible.builtin.include_role:
        name: t4d.WebServerSetup.traefik

    - name: Deploy Authentik identity provider
      ansible.builtin.include_role:
        name: t4d.WebServerSetup.authentik

    - name: Deploy OpenCPU analytics server
      ansible.builtin.include_role:
        name: t4d.WebServerSetup.opencpu
```

### Inventory File

```ini
# inventory.ini
[all]
myserver.com ansible_ssh_user=ansible ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

Adapt the following values to your setup:

- `myserver.com`: The hostname or IP address of your target server.
- `ansible`: The SSH user (set up via the [init_ansible](../roles/init_ansible/README.md) role).
- `~/.ssh/id_ed25519`: Path to your SSH private key.

### Vault File

Create the vault file:

```
ansible-vault create vault.yml
```

Add the following content, adapting the values:

```yaml
# Traefik TLS certificate resolver credentials
# See https://doc.traefik.io/traefik/https/acme/#providers for your provider's variables
CERTRESOLVER:
  infomaniak:
    INFOMANIAK_ACCESS_TOKEN: "your-infomaniak-api-token"
    INFOMANIAK_ENDPOINT: "https://api.infomaniak.com"

# Traefik environment variables (passed to the container)
TRAEFIK_ENV:
  INFOMANIAK_ACCESS_TOKEN: "your-infomaniak-api-token"

# Traefik dashboard users (basic auth)
HTPASSWD_USERS:
  - name: admin
    pw_hash: "$apr1$XXXXXXXXXXXXXXXXXX"

# Authentik configuration
AUTHENTIK_ENV:
  POSTGRES_USER: "authentik"
  POSTGRES_PASSWORD: "a-strong-database-password"
  POSTGRES_DB: "authentik"
  AUTHENTIK_SECRET_KEY: "a-long-random-secret-key"
  AUTHENTIK_TAG: "2025.6.3"
```

To edit the vault later:

```
ansible-vault edit vault.yml
```

#### Generating Traefik Dashboard Credentials

Generate a password hash for the Traefik dashboard:

```
htpasswd -nbB admin
```

Use the hash (everything after the `:`) as the `pw_hash` value.

#### Generating the Authentik Secret Key

Generate a random secret key:

```
openssl rand -base64 48
```

## What Gets Deployed

After running the playbook, your server will have:

1. **System tuning**: A 2GB swap file and optimized kernel parameters for server workloads.
2. **Docker**: Docker engine and Docker Compose installed, with the `dancer` user added to the Docker group.
3. **Traefik** (`https://myserver.com`): Reverse proxy handling all incoming HTTP/HTTPS traffic, with automatic Let's Encrypt TLS certificates via DNS challenge.
4. **Authentik** (`https://auth.myserver.com`): Identity provider with PostgreSQL database, Redis cache, background worker, and automated daily database backups.
5. **OpenCPU** (`https://r.myserver.com`): R computing environment accessible via REST API, secured behind Authentik authentication.

All services communicate through a shared Docker `proxy` network and are fronted by Traefik for TLS termination and routing.

## Used Roles

- [Basic Configuration](../roles/basic_config/README.md)
- [Docker](../roles/docker/README.md)
- [Traefik](../roles/traefik/README.md)
- [Authentik](../roles/authentik/README.md)
- [OpenCPU](../roles/opencpu/README.md)
