# Ansible Role: Install Web App

[![build](https://img.shields.io/github/actions/workflow/status/t4d-gmbh/WebServerSetup/molecule-installWebApp.yml?label=build)](https://github.com/t4d-gmbh/WebServerSetup/actions/workflows/molecule-installWebApp.yml)

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
- `git_repository_branch`: 28-allow-to-run-detached-tasks
- `db_name`: The name of the database for the application.
- `db_user`: The database user for the application.
- `vault_db_password`: The password for the database user (should be stored securely).
- `vault_email_config`: A dictionary with the django email configuration.
  It should be stored securely in a vault file (hence the `vault_` prefix).
  For an example with the django default values, see `/defautls/main.yml`.
- `log_level`: Global log level for all app loggers. Valid values: `DEBUG`, `INFO`, `WARNING`, `ERROR`, `CRITICAL`. Defaults to `WARNING`.
- `log_file`: Optional absolute path to a log file (e.g. `/var/log/myapp/django.log`). Leave empty (default) for console-only logging via stderr. When set, the role creates the parent directory, assigns it to the `django` user, and deploys a logrotate config that rotates the file daily (14 compressed copies retained).
- `app_log_levels`: Optional list of single-key dicts to override the log level per Django app. The key is the app name (lowercased), the value is the log level. Emits `LOG_LEVEL_<APPNAME>=<LEVEL>` entries in the `.env` file. Defaults to `[]` (no overrides; all apps inherit `log_level`). Example:
  ```yaml
  app_log_levels:
    - backends: DEBUG
    - bayesian_networks: INFO
  ```
- `django_project_subdir`: Subdirectory within `/opt/<app_name>` that contains `manage.py`. Defaults to `"."` (repository root — standard Django layout). Set to `"{{ app_name | lower }}"` for the legacy layout where `manage.py` lives inside a subdirectory named after the application.
- `env_extra_vars`: Optional dict of additional plain (non-secret) key/value pairs to append to the `.env` file. Defaults to `{}`. Example:
  ```yaml
  env_extra_vars:
    THIRD_PARTY_API_URL: "https://api.example.com"
    FEATURE_FLAG: "true"
  ```
- `vault_env_extra_vars`: Optional dict of secret key/value pairs to append to the `.env` file. Should be stored in Ansible Vault. Merged with `env_extra_vars`; vault values win on key conflicts. Defaults to `{}`. Example:
  ```yaml
  vault_env_extra_vars:
    THIRD_PARTY_API_KEY: "supersecret"
  ```
  > **Note:** Values must not contain single quotes. Values with spaces or shell special characters (`$`, `#`, `&`, etc.) are safe — they are wrapped in single quotes in the rendered `.env` file.

#### Migration Note — `django_project_subdir`

The working directory for `manage.py` calls is now controlled by `django_project_subdir` (default `"."`). **Existing deployments** where `manage.py` lives inside a `<app_name>/` subdirectory must set in `group_vars/all/main.yml`:

```yaml
django_project_subdir: "{{ app_name | lower }}"
```

This single setting covers `installWebApp`, `updateWebApp`, `gunicornSetup`, and `celerySetup`.

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
7. **Create .env File**: Sets up a `.env` file for the Django application with database, secret key, logging configuration, and any extra variables.
8. **Ensure Log Directory Exists**: Creates the parent directory of `log_file` (owned by `django`) when `log_file` is set.
9. **Deploy logrotate config for Django log**: Installs a logrotate config under `/etc/logrotate.d/` to rotate `log_file` daily (14 compressed copies). Only deployed when `log_file` is set. Uses `copytruncate` because Django keeps the log file descriptor open.
10. **Ensure Static and Media Folders Exist**: Creates directories for static and media files.
10. **Set Permissions**: Configures permissions for the application directories and files.
11. **Install ACL Package**: Ensures the ACL package is installed for managing permissions.
12. **Create Virtual Environment**: Sets up a Python virtual environment for the application.
13. **Install Requirements**: Installs the required Python packages from `pyproject.toml` (preferred) or `requirements.txt`. Fails if neither is present.

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
    git_repository_branch: main
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
