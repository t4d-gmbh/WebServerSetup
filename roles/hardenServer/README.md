# Ansible Role: Harden Server

This Ansible role hardens an Ubuntu server by implementing security best practices. It configures the Uncomplicated Firewall (UFW) with a default-deny incoming policy, hardens SSH configuration, installs fail2ban for intrusion prevention, and conditionally sets up firewall rules for selected services.

## Requirements

- Ansible 2.9 or higher
- Ubuntu server
- Sudo privileges on the target server
- SSH key-based authentication must be set up **before** applying this role (the `init_ansible` role handles this), as it disables root login, challenge-response authentication, and PAM

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `enable_services` | `["ssh"]` | List of services to install and configure UFW rules for. Valid values: `ssh`, `nginx`, `samba`, `nfs`, `tailscale`. |

## Service Details

Each entry in `enable_services` triggers installation of the corresponding package and opens its required firewall ports:

| Service | Package Installed | Firewall Rule |
|---|---|---|
| `ssh` | `openssh-server` | Allow `OpenSSH` UFW application profile |
| `nginx` | `nginx` | Allow `Nginx Full` UFW application profile (HTTP + HTTPS) |
| `samba` | `samba` | Allow `Samba` UFW application profile |
| `nfs` | `nfs-common` | Allow TCP/UDP port 2049 |
| `tailscale` | `tailscale` (from official repo) | Allow UDP/41641 and TCP/443 |

## Dependencies

This role does not depend on other roles, but it is recommended to run the `init_ansible` role first to set up SSH key-based authentication and disable password authentication.

## Tasks Overview

1. **Update and Upgrade Packages**: Runs `apt dist-upgrade` to ensure the system is up to date.
2. **Install Security Packages**: Installs `ufw` and `fail2ban`.
3. **Configure UFW**: Sets default policy to deny incoming and allow outgoing traffic, then enables UFW.
4. **Enable UFW Logging**: Turns on firewall logging for monitoring.
5. **Start and Enable fail2ban**: Ensures fail2ban is running and enabled on boot.
6. **Configure Services**: For each service in `enable_services`, installs the package and opens the required firewall ports.
7. **Harden SSH**: When `ssh` is in `enable_services`, disables root login (`PermitRootLogin no`), challenge-response authentication, and PAM in `/etc/ssh/sshd_config`. Password authentication is handled separately by the `init_ansible` role.

## Example Playbook

```yaml
- hosts: webservers
  become: yes
  vars:
    enable_services:
      - ssh
      - nginx
  roles:
    - t4d.WebServerSetup.hardenServer
```

## Notes

- **SSH access**: Ensure you have working SSH key-based access before running this role. The combination of UFW default-deny and SSH hardening will lock you out if key-based auth is not configured.
- **Nginx Full**: The `nginx` service option opens both port 80 (HTTP) and port 443 (HTTPS) via the `Nginx Full` UFW application profile. If you need HTTPS only, adjust the firewall rules manually after running the role.
- **Tailscale**: The Tailscale repository and GPG key are added automatically. The repository URL is derived from `ansible_distribution` and `ansible_distribution_release` facts, so the target must be a supported distribution.
- **fail2ban**: The role installs and starts fail2ban with its default configuration, which includes an SSH jail on Ubuntu. Custom jail configuration can be added separately.

## License

This role is licensed under the GNU GPLv3 License.

## Author Information

This role was created in 2024 by Jonas I Liechti @ T4D.ch.
