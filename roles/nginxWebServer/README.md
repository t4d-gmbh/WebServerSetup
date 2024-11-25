# Ansible Role: Nginx Web Server

This Ansible role installs and configures Nginx as a web server for a Django application. It sets up the necessary Nginx configuration to serve the application and manage SSL certificates.

## Requirements

- Ansible 2.9 or higher
- Access to a server with `apt` package manager
- A valid SSL certificate and key

## Role Variables

- `app_name`: The name of your application (used for naming the Nginx configuration).
- `server_name`: The domain name or IP address for the server (e.g., `example.com`).
- `cert_path`: The path to the SSL certificate file.
- `key_path`: The path to the SSL certificate key file.
- `static_root`: The path to the static files directory.
- `media_root`: The path to the media files directory.

## Dependencies

This role does not have any external dependencies.

## Installation

To use this role, add it to your Ansible playbook as follows:

```yaml
- hosts: your_target_hosts
  roles:
    - nginxWebServer
```

## Tasks Overview

1. **Install Nginx**: Installs the Nginx web server.
2. **Create Nginx Configuration**: Generates the Nginx configuration file for the Django application using a Jinja2 template.
3. **Enable Nginx Configuration**: Reloads Nginx to apply the new configuration.
4. **Create Symbolic Link to Enable Site**: Links the configuration file from `sites-available` to `sites-enabled`.
5. **Remove Default Nginx Site**: Deletes the default Nginx site configuration to avoid conflicts.

## Usage

1. Define the required variables in your playbook or inventory.
2. Run the playbook to apply the role.

## Example Playbook

```yaml
- hosts: webservers
  become: yes
  vars:
    app_name: "my_django_app"
    server_name: "example.com"
    cert_path: "/etc/letsencrypt/live/example.com/fullchain.pem"
    key_path: "/etc/letsencrypt/live/example.com/privkey.pem"
    static_root: "/opt/my_django_app/static"
    media_root: "/opt/my_django_app/media"
  roles:
    - nginxWebServer
```

## License

This role is licensed under the GNU GPLv3 License.

## Author Information

This role was created in 2024 by Jonas I Liechti @ T4D.ch.
