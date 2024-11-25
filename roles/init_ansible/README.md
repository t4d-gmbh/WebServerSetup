# Ansible Role: Init Ansible

This Ansible role sets up a new user for Ansible automation. It creates a user, configures sudo access, sets up SSH keys, and disables password authentication for enhanced security.

## Requirements

- Ansible 2.9 or higher
- Access to a server with `sudo` privileges

## Role Variables

- `new_ansible_user`: The name of the new user to be created for Ansible automation.
- `public_key`: The public SSH key to be added to the new user's `authorized_keys`.

## Dependencies

This role does not have any external dependencies.

## Installation

To use this role, add it to your Ansible playbook as follows:

```yaml
- hosts: your_target_hosts
  roles:
    - init_ansible
```

## Tasks Overview

1. **Create Ansible User**: Creates a new user for Ansible automation.
2. **Add User to Sudo Group**: Adds the new user to the `sudo` group for administrative privileges.
3. **Allow User to Run Sudo Without a Password**: Configures the sudoers file to allow the user to run commands without a password.
4. **Create .ssh Directory**: Sets up the `.ssh` directory for the new user with appropriate permissions.
5. **Add Public Key to Authorized Keys**: Adds the specified public key to the new user's `authorized_keys` file for SSH access.
6. **Set Permissions for Authorized Keys**: Ensures the correct permissions are set for the `authorized_keys` file.
7. **Disable Password Authentication in SSH Config**: Modifies the SSH configuration to disable password authentication.
8. **Restart SSH Service**: Restarts the SSH service to apply the configuration changes.

## Handlers Overview

- **Restart SSH Service**: Restarts the SSH service when the configuration changes.

## Usage

1. Define the required variables in your playbook or inventory.
2. Run the playbook to apply the role.

## Example Playbook

```yaml
- hosts: all
  become: yes
  vars:
    new_ansible_user: "ansible_user"
    public_key: "ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAr... your_key_here"
  roles:
    - init_ansible
```

## License

This role is licensed under the GNU GPLv3 License.

## Author Information

This role was created in 2024 by Jonas I Liechti @ T4D.ch.
