# Ansible Role: Install Web App

This Ansible role installs and configures a web application, specifically designed for Django applications. It sets up the necessary environment, creates a dedicated user, clones the application repository, and manages application secrets.

## Requirements

- Ansible 2.9 or higher
- Python 3.x
- Access to a server with `apt` package manager
- Git installed on the server
- A valid Git repository for the application

### Django Application

The Django application to install must be configured to use `python-decouple` and expect the following parameters to be deined in the `.env` file:

- `DATABASE_NAME`
- `DATABASE_USER`
- `DATABASE_PASSWORD`
- `SECRET_KEY`
- `ALLOWED_HOSTS`
- `STATIC_ROOT`
- `MEDIA_ROOT`

## Role Variables

- `app_name`: The name of your application (used for directory naming).
- `vault_repository_token`: The token for accessing the Git repository (should be stored securely, e.g., in Ansible Vault).
- `git_remote`: The remote Git server URL.
- `git_repository_path`: The path to the Git repository.
- `db_name`: The name of the database for the application.
- `db_user`: The database user for the application.
- `vault_db_password`: The password for the database user (should be stored securely).

## Dependencies

This role does not have any external dependencies.

## Installation

To use this role, add it to your Ansible playbook as follows:

```yaml
- hosts: your_target_hosts
  roles:
    - installWebApp
```

## Tasks Overview

1. **Install Required Packages**: Installs necessary packages like `git`, `python3-pip`, and `python3-venv`.
2. **Ensure Application Directory Exists**: Creates the application directory with appropriate permissions.
3. **Create a Dedicated User**: Sets up a user for running the Django application.
4. **Clone the Repository**: Clones the application code from the specified Git repository.
5. **Ensure Secret Directory Exists**: Creates a directory for storing application secrets.
6. **Manage Secret Key**: Generates a new secret key if one does not exist and stores it securely.
7. **Create .env File**: Sets up a `.env` file for the Django application with database and secret key information.
8. **Ensure Static and Media Folders Exist**: Creates directories for static and media files.
9. **Set Permissions**: Configures permissions for the application directories and files.
10. **Install ACL Package**: Ensures the ACL package is installed for managing permissions.
11. **Create Virtual Environment**: Sets up a Python virtual environment for the application.
12. **Install Requirements**: Installs the required Python packages from `requirements.txt`.

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
    db_name: "mydatabase"
    db_user: "dbuser"
    vault_db_password: "your_db_password"
  roles:
    - installWebApp
```

## License

This role is licensed under the GNU GPLv3 License.

## Author Information

This role was created in 2024 by Jonas I Liechti @ T4D.ch.
