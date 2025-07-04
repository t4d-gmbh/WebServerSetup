# Headscale VPN Setup 🌐

This README provides instructions on how to use three Ansible roles: **Docker**, **Traefik**, and **Headscale**. These roles are designed to set up a Docker environment with Traefik as a reverse proxy and Headscale for managing Tailscale nodes.

The setup in this example is heavily inspired by the excellent [Headscale VPN video tutorial](https://www.youtube.com/watch?v=DQ1W5JFGBpY) (in German) from [Navigio](https://www.youtube.com/@Navigio1). If you're new to Headscale, Tailscale, and overlay networks (and understand German), this video tutorial is a great starting point. It will help you better understand what the **Docker**, **Traefik**, and **Headscale** roles do in this automated setup.

## Prerequisites

- Ansible installed on your control machine.
- Access to a target machine (e.g., an Ubuntu server) where the roles will be applied.
- SSH access to the target machine using an SSH key.

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
    version: main
```

To install the requirements, simply run:

```
ansible-galaxy install -r requirements.yml
```

## Usage

In short, you need:

- A playbook that installs the roles **docker**, **headscale**, and **traefik** on your target machine.
- An `inventory.ini` file that specifies the target machine and how to access it.
- A `vault.yml` file that contains all the necessary parameters.

To run the playbook and deploy your Headscale server, use the following command:

```
ansible-playbook playbook.yml -i inventory.ini --ask-vault-pass
```

If you adapt the example `playbook.yml` provided below, this command will prompt you for the vault password to decrypt `vault.yml`, set up the Headscale server, and finally create a new file called `api_key.txt` with the API key to connect to your Headscale admin web GUI next to the `playbook.yml`.

### Playbook Example

Here’s an example of the playbook file (`playbook.yml`):

```yaml
---
- name: Deploy Headscale behind Traefik
  hosts: all
  become: true
  vars_files:
    - vault.yml  # Load variables from the vault
  vars:
    dancer_user: "dancer"
    server_url: "https://headscale.myserver.net"
    email: "admin@example.com"  # Email for ACME
    dns_provider: "infomaniak"  # DNS provider
    prefixes_v4: "100.64.0.0/10"  # Default IPv4 prefix
    prefixes_v6: "fd7a:115c:a1e0::/48"  # Default IPv6 prefix
    api_key_expiration: "3000d"  # 3000 days, that's a long time!
    generate_new_api_key: true  # If set to false, no new API key will be generated
    headscale_database_file: "/path/to/my/db.sqlite"  # if not provided a new database is initiated

  tasks:
    - name: Ensure Docker is set up and running
      include_role:
        name: t4d.WebServerSetup.docker

    - name: Create Docker network 'proxy'
      community.docker.docker_network:
        name: proxy
        state: present

    - name: Ensure Headscale is configured
      include_role:
        name: t4d.WebServerSetup.headscale

    - name: Ensure Traefik is configured
      include_role:
        name: t4d.WebServerSetup.traefik
```

### Inventory File

An example of an inventory file (`inventory.ini`) is as follows:

```
[all]
headscale.myserver.net ansible_ssh_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

The following values are examples and need to be adapted to your specific settings:

- `headscale.myserver.net`: The server URL to deploy the roles to.
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
HTPASSWD_USERS:
  - name: alice
    pw_hash:  $apr1$XXXXXXXXXXXXXXXXXX
```

Under `CERTRESOLVER`, you can place all the necessary variables so that Traefik can request and renew the certificate from your specific provider. A list of providers and required variables can be found in [Traefik's official documentation](https://doc.traefik.io/traefik/https/acme/#providers). 

The example provided here is for Infomaniak.

Once the file is ready, simply save and close it.

If you need to further edit the existing `vault.yml`, use:

```
ansible-vault edit vault.yml
```

#### Generating User and Password Hashes

To generate user and password hashes for the `HTPASSWD_USERS` variable, you can use the `htpasswd` command:

```
htpasswd -nbB alice
```

This command will prompt you for a password for the user `alice` and then output a line like:

```
alice:$2y$05$XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

Use the generated hash (i.e., everything after the `:`) in your `vault.yml` file. 🔑


## Used Roles

Here can find detailed information about the roles used in this examples:

- [Docker](../roles/docker/README.md)
- [Traefik](../roles/traefik/README.md)
- [Headscale](../roles/headscale/README.md)
