# Ansible Role: Traefik

This Ansible role installs and configures Traefik, a modern reverse proxy and load balancer. It sets up the necessary configuration files, creates a Docker Compose file for running Traefik, and ensures that the Traefik container is started and managed properly.

## Requirements

- Ansible 2.9 or higher
- Access to a server with `apt` package manager
- Docker and Docker Compose installed
- A user with permissions to manage Docker containers

## Role Variables

- `dancer_user`: The username of the user who will own the Traefik files and directories.
- `traefik.version`: The version of Traefik to install (default: "3.4.3").
- `dns_provider`: The DNS provider for Let's Encrypt certificate resolution.
- `email`: The email address for Let's Encrypt notifications.
- `HTPASSWD_USERS`: A list of users for basic authentication, each containing `name` and `password`.

## Dependencies

This role requires the Docker role to be executed prior to this role.

## Installation

To use this role, add it to your Ansible playbook as follows:

```yaml
- hosts: your_target_hosts
  roles:
    - traefik
```

## Tasks Overview

1. **Create Traefik Configuration Directory**: Creates the directory for Traefik configuration files.
2. **Create Traefik Configuration File**: Generates the Traefik configuration file from a template.
3. **Create Traefik Docker Compose File**: Generates the Docker Compose file for running Traefik.
4. **Create .env File for Traefik**: Sets up an environment file for Traefik configuration.
5. **Create Traefik Users Directory**: Creates a directory for storing user credentials.
6. **Create the Users File**: Generates a file containing user credentials for basic authentication.
7. **Ensure ACL Package is Installed**: Installs the ACL package for managing access control lists.
9. **Start Traefik Container**: Uses Docker Compose to start the Traefik container.

## Usage

1. Define the required variables in your playbook or inventory.
2. Run the playbook to apply the role.

## Example Playbook

```yaml
- hosts: webservers
  become: yes
  vars:
    dancer_user: "your_username"
    dns_provider: "your_dns_provider"
    email: "your_email@example.com"
    HTPASSWD_USERS:
      - name: "admin"
        password: "your_password"
  roles:
    - traefik
```

## License

This role is licensed under the GNU GPLv3 License.

## Author Information

This role was created in 2025 by Jonas I Liechti @ T4D.ch.
