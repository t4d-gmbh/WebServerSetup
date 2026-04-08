# Django Web Application Stack

This example provides two playbooks for deploying and maintaining a Django web application:

1. **Setup playbook** -- Full server provisioning: server hardening, PostgreSQL database, SSL certificates via Certbot, Nginx reverse proxy, Django application installation, Celery task queue with RabbitMQ, and Gunicorn WSGI server.
2. **Update playbook** -- Pulls the latest code from Git, installs dependencies, collects static files, runs migrations, and restarts services.

## Prerequisites

- Ansible installed on your control machine.
- Access to a target machine (e.g., an Ubuntu server) where the roles will be applied.
- SSH access to the target machine using an SSH key (see the [Initial Setup](../README.md#initial-setup) example).
- A domain name with DNS pointing to your server.
- A DNS provider API token for SSL certificate issuance (the example uses Infomaniak via Certbot).
- A Git repository containing your Django application, accessible via HTTPS with a token.

### Django Application Requirements

Your Django application must be set up to use [`python-decouple`](https://pypi.org/project/python-decouple/) (or similar) and read the following variables from a `.env` file:

| Variable | Description |
|---|---|
| `DATABASE_NAME` | PostgreSQL database name |
| `DATABASE_USER` | PostgreSQL user |
| `DATABASE_PASSWORD` | PostgreSQL password |
| `DATABASE_HOST` | Database host (default: `localhost`) |
| `DATABASE_PORT` | Database port (default: `5432`) |
| `DATABASE_ENGINE` | Django database backend |
| `SECRET_KEY` | Django secret key |
| `ALLOWED_HOSTS` | Comma-separated list of allowed hosts |
| `STATIC_ROOT` | Path to collected static files |
| `MEDIA_ROOT` | Path to uploaded media files |
| `EMAIL_HOST` | SMTP server hostname |
| `EMAIL_PORT` | SMTP server port |
| `EMAIL_USE_TLS` | Whether to use TLS for email |
| `EMAIL_HOST_USER` | SMTP username |
| `EMAIL_HOST_PASSWORD` | SMTP password |
| `DEFAULT_FROM_EMAIL` | Default sender email address |

The application must include either a `pyproject.toml` or a `requirements.txt` at the repository root. If both are present, `pyproject.toml` takes priority.

## Directory Structure

```
project/
├── setup.yml        # Initial server provisioning
├── update.yml       # Application updates
├── vault.yml        # Encrypted secrets
├── requirements.yml # Ansible collection dependencies
└── inventory.ini    # Target server(s)
```

## Installation

Create a `requirements.yml` file:

```yaml
---
collections:
  - name: t4d.WebServerSetup
    type: git
    source: https://github.com/t4d-gmbh/WebServerSetup.git
    version: main
```

Install the collection:

```
ansible-galaxy install -r requirements.yml
```

## Setup Playbook

This playbook provisions the server from scratch. Run it once for the initial deployment.

```yaml
# setup.yml
---
- name: Deploy Django Web Application
  hosts: all
  become: true
  vars_files:
    - vault.yml
  vars:
    # Application
    app_name: myDjangoApp

    # Database
    db_name: my_django_db
    db_user: my_django_user

    # Domain and SSL
    server_name: mysite.com
    domain_name: mysite.com
    certbot_email: admin@mysite.com
    cert_path: "/etc/letsencrypt/live/{{ domain_name | lower }}/fullchain.pem"
    key_path: "/etc/letsencrypt/live/{{ domain_name | lower }}/privkey.pem"

    # Git repository
    git_remote: "gitlab.com/"        # or "github.com/"
    git_repository_path: "myorg/myapp.git"
    git_repository_branch: main      # branch or tag

    # Gunicorn
    gunicorn_workers: 3

    # Frontend build (optional -- remove buildFrontend from roles if not needed)
    # Set to the Node.js major version required by your project
    node_major_version: 22

    # Celery (optional -- remove celerySetup from roles if not needed)
    celery_concurrency: 4

    # Server hardening
    enable_services:
      - ssh
      - nginx

    # Swap (optional -- set to null or remove basic_config to skip)
    swapfile_size: "2G"

  roles:
    - t4d.WebServerSetup.basic_config
    - t4d.WebServerSetup.hardenServer
    - t4d.WebServerSetup.postgresqlSetup
    - t4d.WebServerSetup.certbot
    - t4d.WebServerSetup.nginxWebServer
    - t4d.WebServerSetup.installWebApp
    - t4d.WebServerSetup.buildFrontend    # Build Vite/Tailwind frontend assets
    - t4d.WebServerSetup.celerySetup
    - t4d.WebServerSetup.gunicornSetup
```

> **Note:** The `buildFrontend` role is optional. If your Django application does not have a `package.json` or npm-based frontend build step, remove it from the roles list.

Run the setup:

```
ansible-playbook setup.yml --ask-vault-pass -i inventory.ini
```

## Update Playbook

This playbook updates an already-deployed application. It pulls the latest code, installs new dependencies, collects static files, runs migrations, and restarts Gunicorn, Celery, and Nginx.

```yaml
# update.yml
---
- name: Update Django Web Application
  hosts: all
  become: true
  vars_files:
    - vault.yml
  vars:
    app_name: myDjangoApp
    git_remote: "gitlab.com/"
    git_repository_path: "myorg/myapp.git"
    git_repository_branch: main   # branch or tag to deploy
  roles:
    - t4d.WebServerSetup.updateWebApp
    - t4d.WebServerSetup.buildFrontend    # Rebuild frontend assets after code update
```

Run the update:

```
ansible-playbook update.yml --ask-vault-pass -i inventory.ini
```

To deploy a specific branch or tag:

```
ansible-playbook update.yml --ask-vault-pass -i inventory.ini -e "git_repository_branch=v2.1.0"
```

## Inventory File

```ini
# inventory.ini
[all]
mysite.com ansible_ssh_user=ansible ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

Adapt the following values:

- `mysite.com`: The hostname or IP address of your target server.
- `ansible`: The SSH user (set up via the [init_ansible](../roles/init_ansible/README.md) role).
- `~/.ssh/id_ed25519`: Path to your SSH private key.

## Vault File

Create the vault file:

```
ansible-vault create vault.yml
```

Add the following content, adapting the values:

```yaml
# PostgreSQL database password
vault_db_password: "a-strong-database-password"

# Certbot DNS challenge token (for Infomaniak)
vault_infomaniak_DNS_key: "your-infomaniak-api-token"

# Git repository access token (read-only HTTPS token)
vault_repository_token: "glpat-XXXXXXXXXXXXXXXXXXXX"

# Email configuration (for Django's email sending)
vault_email_config:
  email_host: "smtp.myserver.com"
  email_port: "587"
  email_use_tls: "True"
  email_host_user: "noreply@mysite.com"
  email_host_password: "smtp-password"
  email_default_from: "noreply@mysite.com"
```

To edit the vault later:

```
ansible-vault edit vault.yml
```

### Generating a Git Access Token

**GitLab**: Go to *Settings > Access Tokens* and create a token with `read_repository` scope.

**GitHub**: Go to *Settings > Developer settings > Personal access tokens > Fine-grained tokens* and create a token with `Contents: Read-only` permission for the repository.

Use the token as the `vault_repository_token` value. The role constructs the clone URL as:
```
https://oauth2:<token>@<git_remote><git_repository_path>
```

## What Gets Deployed

After running the setup playbook, your server will have:

1. **System tuning**: Swap file and optimized kernel parameters.
2. **Server hardening**: UFW firewall (deny all, allow SSH + Nginx), fail2ban, SSH hardening (no root login, no keyboard-interactive auth).
3. **PostgreSQL** (Docker container): Database for the Django application, running as a Docker container with persistent storage.
4. **SSL certificates**: Automatically obtained via Certbot using DNS challenge, with a cron job for automatic renewal.
5. **Nginx**: Reverse proxy serving the Django application over HTTPS, with static and media file serving.
6. **Django application**: Cloned from Git, installed in a Python virtualenv at `/opt/<app_name>/`, running as the `django` user, with a generated secret key stored securely at `/etc/<app_name>/secret_key.txt`.
7. **Frontend assets**: Node.js installed, npm dependencies resolved, and Vite/Tailwind CSS compiled into production-ready hashed assets collected into `STATIC_ROOT`.
8. **RabbitMQ** (Docker container): Message broker for Celery task processing.
9. **Celery**: Background task worker running as a systemd service.
10. **Gunicorn**: WSGI server running as a systemd service with a Unix socket, proxied by Nginx.

### Service Architecture

```
Client -> Nginx (HTTPS :443) -> Gunicorn (Unix socket) -> Django
                                                            |
                                                            +-> PostgreSQL (Docker :5432)
                                                            +-> RabbitMQ (Docker :5672) -> Celery worker
```

## Operational Notes

### Viewing Logs

```bash
# Gunicorn
sudo journalctl -u gunicorn --follow

# Celery
sudo tail -f /var/log/celery/worker1.log

# Nginx
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log

# PostgreSQL (Docker)
sudo docker logs postgresql --follow
```

### Restarting Services

```bash
sudo systemctl restart gunicorn
sudo systemctl restart celery
sudo systemctl restart nginx
```

### Running Django Management Commands

```bash
sudo -u django /opt/myDjangoApp/venv/bin/python /opt/myDjangoApp/mydjangaapp/manage.py shell
sudo -u django /opt/myDjangoApp/venv/bin/python /opt/myDjangoApp/mydjangaapp/manage.py createsuperuser
```

### SSL Certificate Renewal

Certificates are renewed automatically via a cron job at 2:00 AM daily. To manually renew:

```bash
sudo certbot renew
sudo systemctl reload nginx
```

## Used Roles

- [Basic Configuration](../roles/basic_config/README.md) -- Swap and kernel tuning
- [Harden Server](../roles/hardenServer/README.md) -- Firewall, fail2ban, SSH hardening
- [PostgreSQL Setup](../roles/postgresqlSetup/README.md) -- PostgreSQL in Docker
- [Certbot](../roles/certbot/README.md) -- SSL certificate management
- [Nginx Web Server](../roles/nginxWebServer/README.md) -- Reverse proxy configuration
- [Install Web App](../roles/installWebApp/README.md) -- Django application setup
- [Build Frontend](../roles/buildFrontend/README.md) -- Node.js + Vite/Tailwind frontend build
- [Celery Setup](../roles/celerySetup/README.md) -- Celery + RabbitMQ
- [Gunicorn Setup](../roles/gunicornSetup/README.md) -- Gunicorn WSGI server
- [Update Web App](../roles/updateWebApp/README.md) -- Application updates
