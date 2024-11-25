# Ansible Role: Gunicorn Setup

This Ansible role sets up Gunicorn as a WSGI server for your web application. It creates a Python virtual environment, installs Gunicorn, and configures it to run as a systemd service.

## Requirements

- Ansible 2.9 or higher
- Python 3.x
- Access to a server with `pip` and `systemd`
- A web application compatible with Gunicorn (e.g., Django, Flask)

## Role Variables

- `app_name`: The name of your application (used for directory and service naming).
- `gunicorn_workers`: The number of Gunicorn worker processes to spawn.

## Dependencies

This role does not have any external dependencies.

## Installation

To use this role, add it to your Ansible playbook as follows:

```yaml
- hosts: your_target_hosts
  roles:
    - gunicornSetup
```

## Tasks Overview

1. **Create a Python Virtual Environment**: Sets up a virtual environment for the web application.
2. **Install Gunicorn**: Installs Gunicorn in the created virtual environment.
3. **Create Gunicorn systemd Service File**: Configures a systemd service for Gunicorn to manage the application.
4. **Enable and Start Gunicorn Service**: Enables and starts the Gunicorn service to run on system boot.

## Handlers Overview

- **Reload Gunicorn**: Reloads the Gunicorn service when the configuration changes.

## Usage

1. Define the required variables in your playbook or inventory.
2. Run the playbook to apply the role.

## Example Playbook

```yaml
- hosts: webservers
  become: yes
  vars:
    app_name: "my_web_app"
    gunicorn_workers: 3
  roles:
    - gunicornSetup
```

## License

This role is licensed under the MIT License.

## Author Information

This role was created in 2024 by Jonas I Liechti @ T4D.ch.
