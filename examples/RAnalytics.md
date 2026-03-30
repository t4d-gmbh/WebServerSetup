# R Analytics Stack

This example deploys an authenticated R computing environment: Docker engine, Authentik identity provider for user management, and OpenCPU for running R scripts via a REST API. All services are connected through a shared Docker network and can be fronted by an existing Traefik reverse proxy.

## Prerequisites

- Ansible installed on your control machine.
- Access to a target machine (e.g., an Ubuntu server) where the roles will be applied.
- SSH access to the target machine using an SSH key (see the [Initial Setup](../README.md#initial-setup) example).
- A domain name with DNS pointing to your server.
- A DNS provider API token for TLS certificate issuance (used by Traefik labels in the Docker Compose configuration).
- An existing Traefik instance or the [Full Infrastructure Stack](FullInfrastructure.md) which includes Traefik.

## Directory Structure

```
project/
├── playbook.yml
├── vault.yml
├── requirements.yml
└── inventory.ini
```

## Installation

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

Run the playbook:

```
ansible-playbook playbook.yml -i inventory.ini --ask-vault-pass
```

### Playbook Example

```yaml
# playbook.yml
---
- name: Deploy R Analytics Stack
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
      # R packages to install in the OpenCPU container
      RUN_R:
        - "install.packages('pak', repos=c(CRAN='https://cran.r-project.org'))"
        # Add your own R packages here, e.g.:
        # - "pak::pkg_install('ggplot2')"
        # - "pak::pkg_install('dplyr')"

  tasks:
    - name: Install and configure Docker
      ansible.builtin.include_role:
        name: t4d.WebServerSetup.docker

    - name: Create Docker proxy network
      community.docker.docker_network:
        name: proxy
        state: present

    - name: Deploy Authentik identity provider
      ansible.builtin.include_role:
        name: t4d.WebServerSetup.authentik

    - name: Deploy OpenCPU analytics server
      ansible.builtin.include_role:
        name: t4d.WebServerSetup.opencpu
```

> **Note:** This playbook assumes an existing Traefik reverse proxy is running and connected to the `proxy` Docker network. If you don't have Traefik set up yet, use the [Full Infrastructure Stack](FullInfrastructure.md) example instead, which includes Traefik.

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

#### Generating the Authentik Secret Key

```
openssl rand -base64 48
```

## What Gets Deployed

After running the playbook, your server will have:

1. **Docker**: Docker engine and Docker Compose installed, with the `dancer` user added to the Docker group.
2. **Authentik** (`https://auth.myserver.com`): Identity provider with PostgreSQL database, Redis cache, background worker, and automated daily database backups. Provides user authentication for OpenCPU.
3. **OpenCPU** (`https://r.myserver.com`): R computing environment accessible via a REST API. Any R package specified in the `RUN_R` list is pre-installed in the container. Access is secured behind Authentik's forward-auth proxy.

## Configuring Authentik for OpenCPU

After deployment, you need to configure Authentik to protect OpenCPU:

1. Log in to Authentik at `https://auth.myserver.com/if/admin/`.
2. Create an **Application** for OpenCPU.
3. Create a **Proxy Provider** with the external URL `https://r.myserver.com`.
4. Assign the provider to the application.
5. Create an **Outpost** (embedded or proxy) and assign the OpenCPU application to it.

Users will then be redirected to Authentik for login before accessing the OpenCPU API.

## Customizing R Packages

To install additional R packages in the OpenCPU container, add entries to the `RUN_R` list:

```yaml
opencpu:
  RUN_R:
    - "install.packages('pak', repos=c(CRAN='https://cran.r-project.org'))"
    - "pak::pkg_install('ggplot2')"
    - "pak::pkg_install('dplyr')"
    - "pak::pkg_install('jsonlite')"
```

The packages are installed during the Docker image build. To update packages, re-run the playbook -- the image will be rebuilt with the updated package list.

## Used Roles

- [Docker](../roles/docker/README.md)
- [Authentik](../roles/authentik/README.md)
- [OpenCPU](../roles/opencpu/README.md)
