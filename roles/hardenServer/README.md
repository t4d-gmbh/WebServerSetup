# Ansible Role: Harden Server

This Ansible role is designed to harden an Ubuntu server by implementing security best practices. It focuses on configuring the Uncomplicated Firewall (UFW), securing SSH access, and installing essential packages for a web server environment.

## Requirements

- Ansible 2.9 or higher
- Ubuntu server (tested on 20.04 and 22.04)
- Sudo privileges on the target server

## Role Variables

This role does not require any specific variables to be set. However, you may want to customize the list of packages installed or modify the SSH configuration settings as needed.

## Dependencies

This role does not have any dependencies on other roles.

## Example Playbook

To use this role, include it in your playbook as follows:

:::yaml
- hosts: your_target_hosts
  become: yes
  roles:
    - hardenServer
:::

Replace `your_target_hosts` with the appropriate inventory group or host.

## Tasks Overview

The following tasks are performed by this role:

1. **Update and Upgrade Packages**: Ensures the system is up to date with the latest packages.
2. **Install Necessary Packages**: Installs Nginx, UFW (Uncomplicated Firewall), and Fail2Ban for intrusion prevention.
3. **Configure UFW**:
   - Enables UFW and allows SSH traffic.
   - Allows HTTPS traffic for Nginx.
   - Denies all other incoming traffic.
4. **Configure SSH**: 
   - Disables password authentication, root login, challenge-response authentication, and PAM.
5. **Restart SSH Service**: Applies the SSH configuration changes.
6. **Enable UFW Logging**: Turns on logging for UFW to monitor firewall activity.

## Security Considerations

- Ensure that you have SSH key-based authentication set up before applying this role, as it disables password authentication.
- Review the installed packages and adjust them according to your specific requirements.
- Test the role in a safe environment before deploying it to production.

## License

This role is licensed under the GNU PGLv3 License.

## Author Information

This role was created in 2024 by Jonas I Liechti @ T4D.ch
