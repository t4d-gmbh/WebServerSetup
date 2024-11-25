# Ansible Role: Update Web App

This Ansible role updates and manages a Django web application. It fetches the latest changes from the Git repository, installs dependencies, collects static files, and applies database migrations.

## Requirements

- Ansible 2.9 or higher
- Python 3.x
- Access to a server with `apt` package manager
- Git installed on the server
- A valid Git repository for the application

## Role Variables

- `app_name`: The name of your application (used for directory naming).
- `vault_repository_token`: The token for accessing the Git repository (should be stored securely, e.g., in Ansible Vault).
- `git_remote`: The remote Git server URL.
- `git_repository_path`: The path to the Git repository.
- `git_repository_branch`: The branch or tag to deploy

## Dependencies

This role does not have any external dependencies.

## Installation

To use this role, add it to your Ansible playbook as follows:

```yaml
- hosts: your_target_hosts
  roles:
    - updateWebApp
```

## Tasks Overview

1. **Ensure ACL Package is Installed**: Installs the ACL package for managing permissions.
2. **Fetch Latest Changes from Git Repository**: Pulls the latest changes from the specified Git repository.
3. **Create Virtual Environment**: Sets up a Python virtual environment for the application if it doesn't already exist.
4. **Install Requirements**: Installs the required Python packages from `requirements.txt` within the virtual environment.
5. **Collect Static Files**: Gathers static files for the Django application.
6. **Apply Database Migrations**: Applies any pending database migrations.

## Handlers Overview

- **Restart Gunicorn Service**: Restarts the Gunicorn service to apply changes.
- **Restart Web Server**: Restarts the web server (e.g., Nginx) to apply changes.

## Usage

1. Define the required variables in your playbook or inventory.
2. Run the playbook to apply the role.

## Example Playbook

```yaml
- hosts: webservers
  become: yes
  vars:
    app_name: "my_django_app"
    vault_repository_token: "your_repository_token"
    git_remote: "https://github.com/your_user/"
    git_repository_path: "your_repo.git"
    git_repository_branch: "main"
  roles:
    - updateWebApp
```

## License

This role is licensed under the GNU GPLv3 License.

## Author Information

This role was created in 2024 by Jonas I Liechti @ T4D.ch.
