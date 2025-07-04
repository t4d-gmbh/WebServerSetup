# Ansible Role: Headscale

This Ansible role installs and configures Headscale, a self-hosted implementation of Tailscale. It sets up the necessary directories, configuration files, and starts the Headscale container using Docker. The role also manages the creation of an API key for the admin interface if required.

## Requirements

- Ansible 2.9 or higher
- Access to a server with `apt` package manager
- Docker and Docker Compose installed
- A user with permissions to manage Docker containers

## Role Variables

- `dancer_user`: The username of the user who will own the Headscale files and directories.
- `headscale.version`: The version of Headscale to install (default: "0.26.1").
- `api_key_expiration`: The expiration time for the API key (default: "365d").
- `generate_new_api_key`: Boolean to determine if a new API key should be generated (default: false).
- `nameservers`: List of DNS nameservers to be used by Headscale.
- `server_url`: The URL of the Headscale server.
- `prefixes_v4`: The IPv4 prefixes for Headscale.
- `prefixes_v6`: The IPv6 prefixes for Headscale.
- `email`: The email address for Let's Encrypt notifications.
- `headscale_database_file`: Optional path to a sqlite database file that will be used as the sqlite db of the headscale instance. If not provided a new database will be initiated.

## Dependencies

This role requires the Docker role to be executed prior to this role.

## Installation

To use this role, add it to your Ansible playbook as follows:

```yaml
- hosts: your_target_hosts
  roles:
    - headscale
```

## Tasks Overview

1. **Ensure User Exists**: Checks that the specified user exists on the system.
2. **Create Headscale Configuration Directory**: Creates the directory for Headscale configuration files.
3. **Create Headscale Data Directory**: Sets up the data directory for Headscale.
4. **Create Headscale Configuration File**: Generates the Headscale configuration file from a template.
5. **Create Headscale Docker Compose File**: Generates the Docker Compose file for running Headscale.
6. **Ensure ACL Package is Installed**: Installs the ACL package for managing access control lists.
7. **Start Headscale Container**: Uses Docker Compose to start the Headscale container.
8. **Create API Key**: Optionally generates an API key for the admin interface and saves it to a file.

## Usage

1. Define the required variables in your playbook or inventory.
2. Run the playbook to apply the role.

## Example Playbook

```yaml
- hosts: webservers
  become: yes
  vars:
    dancer_user: "your_username"
    server_url: "https://example.com"
    prefixes_v4: "100.64.0.0/10"
    prefixes_v6: "fc00::/7"
    email: "your_email@example.com"
    generate_new_api_key: true
  roles:
    - headscale
```

## License

This role is licensed under the GNU GPLv3 License.

## Author Information

This role was created in 2025 by Jonas I Liechti @ T4D.ch.
