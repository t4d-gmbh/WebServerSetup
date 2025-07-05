# Ansible Role: Docker

This Ansible role installs and configures Docker and Docker Compose on an Ubuntu server. It ensures that the necessary system packages are installed, the Docker GPG key and repository are added, and the Docker service is started and enabled. Additionally, it adds a specified user to the Docker group for managing containers without sudo.

## Requirements

- Ansible 2.9 or higher
- Access to a server with `apt` package manager
- Ubuntu 20.04 (Focal Fossa) or later

## Role Variables

- `dancer_user`: The username of the user to be added to the Docker group.
- `DOCKER_LOGIN`: Optional variable that needs to provide `user` and `password`. If defined, then it is used to login to Docker Hub

## Dependencies

This role does not have any external dependencies.

## Installation

To use this role, add it to your Ansible playbook as follows:

```yaml
- hosts: your_target_hosts
  roles:
    - docker
```

## Tasks Overview

1. **Install Required System Packages**: Installs necessary packages such as `apt-transport-https`, `ca-certificates`, `curl`, `software-properties-common`, and `virtualenv`.
2. **Add Docker GPG Key**: Adds the official Docker GPG key to the system.
3. **Add Docker Repository**: Configures the Docker APT repository for installation.
4. **Update APT and Install Docker Packages**: Installs Docker Engine, Docker CLI, containerd, and Docker Compose.
5. **Add User to Docker Group**: Adds the specified user to the Docker group to allow non-sudo access to Docker commands.
6. **Start and Enable Docker Service**: Ensures that the Docker service is started and enabled to run on boot.
7. **Loggin in to Docker Hub**: If credentails are provided they are used to login to Docker Hub.

## Usage

1. Define the required variables in your playbook or inventory.
2. Run the playbook to apply the role.

## Example Playbook

```yaml
- hosts: webservers
  become: yes
  vars:
    dancer_user: "your_username"
    DOCKER_LOGIN:  # NOTE: This should be defined in a vault!
      username: timmy
      password: "timmy's-password"
  roles:
    - docker
```

## License

This role is licensed under the GNU GPLv3 License.

## Author Information

This role was created in 2025 by Jonas I Liechti @ T4D.ch.
