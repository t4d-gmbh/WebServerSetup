# Ansible Roles README

This README provides instructions on how to use three Ansible roles: **Docker**, **Traefik**, and **Headscale**. These roles are designed to set up a Docker environment with Traefik as a reverse proxy and Headscale for managing Tailscale nodes.

## Prerequisites

- Ansible installed on your control machine.
- Access to a target machine (e.g., an Ubuntu server) where the roles will be applied.
- SSH access to the target machine by means of an SSH key

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

You will need the [T4D.WebServerSetup]() ansible collection

The simplest way is to add a `requirements.yml` file to your project  with the following content:
```
---
collections:
  - name: t4d.WebServerSetup
    type: git
    version: main
```
To install the requirements, simply type:

```
ansible-galaxy install -r requirements.yml
```

## Usage

In short, you need:

- A playbook that install the roles _docker_, _headscale_ and _traefik_ on your target machine.
- A `inventory.ini` file that specifies the target machine and how to access it.
- A `vault.yml` file that contains all the necessary parameters.

To run the playbook and deploy your Headscale server, use the following command:

```
ansible-playbook playbook.yml -i inventory.ini --ask-vault-pass
```

If you adapt the example `playbook.yml` provided below, this command will prompt you for the vault password to decrypt `vault.yml`, go ahead and set up the Headscale server for you and finallly leave a new file, called `api_key.txt` next to the `playbook.yml` that contains the API key allowing you to connect the Headscale admin webgui to your headscale instance. 

### Playbook Example

Here is an example playbook that utilizes the three roles:

Under `vars` you must at least adapt `server_url`, `email` and `dns_provider`.
A list of supported providers can be found in [traefik's official documentation](https://doc.traefik.io/traefik/https/acme/#providers).


```
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
    api_key_expiration: "3000d"  # Those are 3000 days, that's a long time!
    generate_new_api_key: true  # If set to false, no new api key will be generated

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
The following values are examplary and need to be adapted to your specific setting:

- `headscale.myserver.net`: The server url to deploy the roles to.
- `ubuntu`: The user that can access the server.
- `~/.ssh/id_ed25519`: Path to the ssh private key to access the server.

### Vault File

To create a `vault.yml` file, you can use the following template:

```
CERTRESOLVER:
  INFOMANIAK_ACCESS_TOKEN: "sNphaL-XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
  INFOMANIAK_ENDPOINT: "https://api.infomaniak.com"
HTPASSWD_USERS:
  - name: alice
    pw_hash:  $apr1$XXXXXXXXXXXXXXXXXX
```
Under `CERTRESOLVER` you can place all the necessary variable so that traefik can request and renew the certificate at your specific provides.
A list of providers and required variables can be found in [traefik's official documentation](https://doc.traefik.io/traefik/https/acme/#providers).

Present here is an example for Infomaniak.


Once the file is ready you can encrypt it using Ansible Vault:

```
ansible-vault encrypt vault.yml
```

Alternatively you can directly open an edit an empty `vault.yml` file with:
```
ansible-vault create vault.yml
```

#### Generating User and Password Hashes

To generate user and password hashes for the `HTPASSWD_USERS` variable, you can use the `htpasswd` command:

```
htpasswd -nbB alice yourpassword
```

This command will output a line like:

```
alice:$2y$05$XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

Use the generated hash in your `vault.yml` file.

## Role Descriptions

### Docker Role

The Docker role installs Docker and its dependencies, adds a specified user to the Docker group, and ensures the Docker service is running.

**Key Tasks:**
- Install required system packages.
- Add Docker GPG key and repository.
- Install Docker and Docker Compose.
- Start and enable the Docker service.

### Traefik Role

The Traefik role sets up Traefik as a reverse proxy, creating necessary configuration files and directories.

**Key Tasks:**
- Create Traefik configuration directory.
- Generate Traefik configuration and Docker Compose files from templates.
- Start the Traefik container.

### Headscale Role

The Headscale role installs and configures Headscale, a self-hosted implementation of Tailscale.

**Key Tasks:**
- Ensure the specified user exists.
- Create Headscale configuration and data directories.
- Generate configuration files and start the Headscale container.

## Running the Playbook

To run the playbook, use the following command:

```
ansible-playbook playbook.yml -i inventory.ini --ask-vault-pass
```

This command will prompt you for the vault password to decrypt `vault.yml`.

## Requirements

- Ensure you have the necessary Ansible collections installed. You can specify them in the `requirements.yml` file and install them using:

```
ansible-galaxy install -r requirements.yml
```

## Inventory File

An example of an inventory file (`inventory.ini`) is as follows:

```
[all]
headscale.myserver.net ansible_ssh_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_ed25519
``` 

This inventory file specifies the target host and the SSH user to connect with. Adjust the values as necessary for your environment.
