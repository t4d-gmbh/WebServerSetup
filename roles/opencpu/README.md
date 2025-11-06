# Ansible Role: OpenCPU

This Ansible role builds, installs and configures an OpenCPU container.
By means of container labels the instance is configured to use authentik as authentication proxy effectively securing the installation from un-authorized access.

## Requirements

- Ansible 2.9 or higher
- Access to a server with `apt` package manager
- Docker and Docker Compose installed
- A user with permissions to manage Docker containers

## Role Variables

- `docker_user`: The username of the user who will own the Headscale files and directories.
- `dns_provider`: 
- `opencpu`: A dictionary holding opencpu specific variables:
  - `DOMAIN`: The domain under which the opencpu service should be reachable.
  - `CONTAINER_NAME`: Name the container; defaults to "opencpu".
  - `DOCKER_IMAGE`: Which OpenCPU container image to use as basis. By default `docker.io/opencpu/ubuntu-24.04` is used.
  - `DOCKER_TAG`: The specific tag to use. By default `v2.2.14-2` is used.
  - `RUN_R`: A list of valid R statements to execute in order when building the docker image.
     
    The entry:
    ```yml
    - "install.packages('pak', repos=c(CRAN='https://cran.r-project.org'))"
    ```
    will be translated to:
    ```
    RUN R -e "install.packages('pak', repos=c(CRAN='https://cran.r-project.org'))"
    ```
    in the Dockerfile.

  - `AUTHENTIK_PROXY_CONTAINER`: _Optional_ name of the docker container running the authentik outpost used to authenticate for the opencpu service.
    If omitted the OpenCPU service is not secured and can be reached without any authentication.

    In case you are unsure what to put here, simply use the container name of the authentik service (see authentik role) and configure configure authentik's forwardAuth procedure with an "embedded Outpost".

## Dependencies

This role requires the Docker role to be executed prior to this role.

## Installation

To use this role, add it to your Ansible playbook as follows:

```yaml
- hosts: your_target_hosts
  roles:
    - opencpu
```

## Tasks Overview

1. **Ensure User Exists**: Checks that the specified user exists on the system.
1. **Ensure ACL Package is Installed**: Installs the ACL package for managing access control lists.
1. **Create OpenCPU Configuration Directory**: Creates the directory for OpenCPU configuration files.
1. **Create OpenCPU Container Directory**: Sets up the OpenCPU custom container directory.
1. **Present OpenCPU container build file**: Deploys the `Dockerfile` with the custom image definition.
1. **Log into DockerHub**: If credentails are provided they are used to login to Docker Hub.
1. **Pull the container image**: Pulls the specified OpenCPU Docker image and notify the **Run OpenCPU docker compose** in case the image changed.
1. **Present OpenCPU docker-compose file**: Deploys the docker compose file and notifies the **Run OpenCPU docker compose** handler in case the file changed. 

## Handlers Overview:

1. **Run OpenCPU docker compose**: Asserts that the containers are up and recreates them in case the configuration changed.

## Usage

1. Define the required variables in your playbook or inventory.
2. Run the playbook to apply the role.

## Example Playbook

```yaml
- hosts: webservers
  become: yes
  vars:
    docker_user: "dancer"
    dns_provider: "infomaniak"
    opencpu:
      DOMAIN: "r.myserver.com"
      AUTHENTIK_PROXY_CONTAINER: "authentik-server"
      CONTAINER_NAME: "opencpu"
      DOCKER_IMAGE: "docker.io/opencpu/ubuntu-24.04"
      DOCKER_TAG: "v2.2.14-2"
    
      RUN_R:
        - "install.packages('pak', repos=c(CRAN='https://cran.r-project.org'))"
        - "pak::repo_add(INLA = 'https://inla.r-inla-download.org/R/stable/')"
  roles:
    - authentik  # recommended in case you want to secure your opencpu service
    - opencpu
```
## License

This role is licensed under the GNU GPLv3 License.

## Author Information

This role was created in 2025 by Jonas I Liechti @ T4D.ch.
