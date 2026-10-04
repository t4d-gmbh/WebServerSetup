# Ansible Role: Docker

[![build](https://img.shields.io/github/actions/workflow/status/t4d-gmbh/WebServerSetup/molecule-docker.yml?label=build)](https://github.com/t4d-gmbh/WebServerSetup/actions/workflows/molecule-docker.yml)

This Ansible role installs and configures Docker Engine and the Docker Compose plugin (V2+) on an Ubuntu server. It ensures that the necessary system packages are installed, the Docker GPG key is placed in `/etc/apt/keyrings` and the repository is added as a deb822 source pinned to the host's release codename, and the Docker service is started and enabled. Additionally, it adds a specified user to the Docker group for managing containers without sudo.

## Requirements

- Ansible 2.9 or higher
- Access to a server with `apt` package manager
- Ubuntu 22.04 (Jammy Jellyfish) or later (apt >= 2.4 for ASCII-armored `Signed-By` keys)

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

1. **Install Required System Packages**: Installs necessary packages such as `ca-certificates`, `curl`, `software-properties-common`, and `virtualenv`.
2. **Remove Legacy Repository Artifacts**: Removes the old `docker.list` source and `trusted.gpg.d/docker.gpg` key written by previous versions of this role (which relied on the deprecated `apt-key`).
3. **Add Docker GPG Key**: Downloads the official Docker GPG key to `/etc/apt/keyrings/docker.asc`.
4. **Add Docker Repository**: Writes a deb822 source (`/etc/apt/sources.list.d/docker.sources`) signed with the keyring and pinned to the host's release codename (e.g. `resolute` on Ubuntu 26.04).
5. **Update APT and Install Docker Packages**: Installs Docker Engine, Docker CLI, containerd, and the Docker Compose plugin (V2+).
6. **Add User to Docker Group**: Adds the specified user to the Docker group to allow non-sudo access to Docker commands.
7. **Start and Enable Docker Service**: Ensures that the Docker service is started and enabled to run on boot.
8. **Loggin in to Docker Hub**: If credentails are provided they are used to login to Docker Hub.

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
