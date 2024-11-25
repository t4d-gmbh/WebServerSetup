# Ansible Role: PostgreSQL Setup

This Ansible role installs and configures PostgreSQL using Docker. It ensures that the necessary packages are installed, Docker is set up, and a PostgreSQL container is created and running.

## Requirements

- Ansible 2.9 or higher
- Access to a server with `apt` package manager
- Docker installed on the server

## Role Variables

- `postgresql_version`: The version of PostgreSQL to install (default: `latest`).
- `postgresql_container_name`: The name of the PostgreSQL container (default: `postgresql`).
- `postgresql_port`: The port on which PostgreSQL will be accessible (default: `5432`).
- `postgresql_data_dir`: The directory on the host where PostgreSQL data will be stored (default: `/var/lib/postgresql/data`).
- `db_name`: The name of the PostgreSQL database to create.
- `db_user`: The PostgreSQL user to create.
- `vault_db_password`: The password for the PostgreSQL user (should be stored securely).

## Dependencies

This role requires the `community.docker` collection. You can install it using:

```bash
ansible-galaxy collection install community.docker
```

## Installation

To use this role, add it to your Ansible playbook as follows:

```yaml
- hosts: your_target_hosts
  roles:
    - postgresqlSetup
```

## Tasks Overview

1. **Update apt Package Index**: Updates the package index to ensure the latest packages are available.
2. **Install Required Packages for Docker**: Installs necessary packages for Docker installation.
3. **Install Docker**: Installs the Docker engine.
4. **Ensure Docker Service is Running**: Starts and enables the Docker service.
5. **Pull PostgreSQL Docker Image**: Pulls the specified version of the PostgreSQL Docker image.
6. **Create PostgreSQL Container**: Creates and starts a PostgreSQL container with the specified environment variables and volume mappings.

## Usage

1. Define the required variables in your playbook or inventory.
2. Run the playbook to apply the role.

## Example Playbook

```yaml
- hosts: database_servers
  become: yes
  vars:
    postgresql_version: "13"  # Specify the desired PostgreSQL version
    postgresql_container_name: "my_postgres"
    postgresql_port: 5432
    postgresql_data_dir: "/var/lib/postgresql/data"
    db_name: "mydatabase"
    db_user: "myuser"
    vault_db_password: "your_secure_password"
  roles:
    - postgresqlSetup
```

## License

This role is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

## Author Information

This role was created in 2024 by Jonas I Liechti @ T4D.ch.
