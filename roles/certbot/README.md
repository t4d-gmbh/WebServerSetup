# Ansible Role: Certbot

This Ansible role installs and configures Certbot for obtaining and managing SSL certificates using the Infomaniak DNS plugin. It sets up a Python virtual environment, installs necessary dependencies, and ensures that SSL certificates are valid and renewed automatically.

## Requirements

- Ansible 2.9 or higher
- Python 3.x
- Access to a server with `apt` package manager
- Infomaniak DNS credentials

## Role Variables

- `vault_infomaniak_DNS_key`: The token for Infomaniak DNS API, should be stored securely (e.g., in Ansible Vault).
- `domain_name`: The domain for which the SSL certificate is requested.
- `cert_path`: The path where the SSL certificate will be stored.
- `key_path`: The path where the private key will be stored.

## Dependencies

This role does not have any external dependencies.

## Installation

To use this role, add it to your Ansible playbook as follows:

```yaml
- hosts: your_target_hosts
  roles:
    - certbot
```

## Tasks Overview

1. **Install Python Dependencies**: Installs `python3-pip`, `python3-venv`, and `python3-packaging`.
2. **Create a Python Virtual Environment**: Sets up a virtual environment for Certbot.
3. **Install Certbot and Plugins**: Installs `certbot`, `certbot-nginx`, and `certbot-dns-infomaniak` in the virtual environment.
4. **Create Symbolic Link for Certbot**: Links the Certbot executable to `/usr/bin/certbot` for easy access.
5. **Ensure Application Directory Exists**: Creates the `/etc/letsencrypt` directory with appropriate permissions.
6. **Create Infomaniak Credentials File**: Copies the Infomaniak DNS credentials to a secure file.
7. **Check SSL Certificate Validity**: Verifies if the existing SSL certificate is valid.
8. **Obtain SSL Certificate**: Requests a new SSL certificate if the existing one is invalid or does not exist.
9. **Debug Output**: Displays the output of the Certbot command for troubleshooting.
10. **Set Up Cron Job for Renewal**: Configures a cron job to automatically renew the SSL certificate daily at 2 AM.

## Usage

1. Define the required variables in your playbook or inventory.
2. Run the playbook to apply the role.

## Example Playbook

```yaml
- hosts: webservers
  become: yes
  vars:
    vault_infomaniak_DNS_key: "your_infomaniak_dns_token"
    domain_name: "example.com"
    cert_path: "/etc/letsencrypt/live/example.com/fullchain.pem"
    key_path: "/etc/letsencrypt/live/example.com/privkey.pem"
  roles:
    - certbot
```

## License

This role is licensed under the GNU GPLv3 License.

## Author Information

This role was created in 2024 by Jonas I Liechti @ T4D.ch.
