# Headscale VPN setup

This README provides instructions on how to use three Ansible roles: **Docker**, **Traefik**, and **Headscale**.

These roles are designed to set up a Docker environment with Traefik as a reverse proxy and Headscale for managing Tailscale nodes.

The setup deployed in this example is heavily inspired from the very nice [Headscale VPN video tutorial](https://www.youtube.com/watch?v=DQ1W5JFGBpY) (in German) from [Navigio](https://www.youtube.com/@Navigio1).
If you are very new to Headscale, Tailscale and overlay networks (and understand German) above video tutorial is a good starting pont and will also help you to better understand what the **Docker**, **Traefik** and **Headscale** roles are doing in this automated setup.

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

- A playbook that installs the roles _docker_, _headscale_ and _traefik_ on your target machine.
- A `inventory.ini` file that specifies the target machine and how to access it.
- A `vault.yml` file that contains all the necessary parameters.

To run the playbook and deploy your Headscale server, use the following command:

```
ansible-playbook playbook.yml -i inventory.ini --ask-vault-pass
```

If you adapt the example `playbook.yml` provided below, this command will prompt you for the vault password to decrypt `vault.yml`, go ahead and set up the Headscale server for you and finallly leave a new file, called `api_key.txt`, with the API key to connect your headscale admin webgui next to the `playbook.yml`.

### Playbook Example

This is an example for the playbook file (`playbook.yml`).

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
The following values are exemplary and need to be adapted to your specific setting:

- `headscale.myserver.net`: The server url to deploy the roles to.
- `ubuntu`: The user that can access the server.
- `~/.ssh/id_ed25519`: Path to the ssh private key to access the server.

### Vault File

To create a `vault.yml` file, you can run the command:

```
ansible-vault create vault.yml
```
Then simply copy/paste and adapt the following content:

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


Once the file is ready simply save and close the file.

If you need to further edit the existing `vault.yml` use:

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

Use the generated hash (i.e. everything after the `:`)  in your `vault.yml` file.
