# Ansible Collection: Webserver🌐 Automation
This Ansible collection provides a set of roles designed to automate the setup and management of web servers, specifically tailored for deploying Django applications. The collection includes roles for user initialization, server hardening, PostgreSQL setup, SSL certificate management, Nginx configuration, web application installation, Gunicorn setup, and application updates, as well as container management with Docker, Headscale, and Traefik.

## Roles Included

*   **[Basic Configuration](roles/basic_config/README.md)** ⚙️
    Manages **foundational server configurations**, primarily focusing on **swap space management** and **kernel parameter tuning** for optimal performance [conversation history].
*   **[Init Ansible](roles/init_ansible/README.md)** 👤
    Sets up a new user for Ansible automation, configures sudo access, sets up SSH keys, and disables password authentication for enhanced security.
*   **[Harden Server](roles/harden_server/README.md)** 🔒
    Hardens an Ubuntu server by implementing security best practices, including configuring the Uncomplicated Firewall (UFW) and securing SSH access.
*   **[PostgreSQL Setup](roles/postgresql_setup/README.md)** 🐘
    Installs and configures PostgreSQL using Docker, ensuring that the necessary packages are installed and a PostgreSQL container is created and running.
*   **[Certbot](roles/certbot/README.md)** 🔑
    Installs and configures Certbot for obtaining and managing SSL certificates using the Infomaniak DNS plugin, ensuring automatic renewal of certificates.
*   **[Nginx Web Server](roles/nginx_web_server/README.md)** 🌍
    Installs and configures Nginx as a web server for a Django application, managing SSL certificates and serving static files.
*   **[Install Web App](roles/install_web_app/README.md)** 📦
    Installs and configures a Django web application, setting up the necessary environment, creating a dedicated user, and managing application secrets.
*   **[Gunicorn Setup](roles/gunicorn_setup/README.md)** 🚀
    Sets up Gunicorn as a WSGI server for your web application, configuring it to run as a systemd service.
*   **[Celery Setup](roles/celery_setup/README.md)** 🍃
    Sets up Celery as a task queue for your web application, configuring RabbitMQ as a message broker and managing Celery as a systemd service.
*   **[Update Web App](roles/update_web_app/README.md)** 🔄
    Updates and manages a Django web application by fetching the latest changes from the Git repository, installing dependencies, collecting static files, and applying database migrations.
*   **[Docker Setup](roles/docker_setup/README.md)** 🐳
    Installs Docker and Docker Compose, configures the Docker service, and adds the specified user to the Docker group for container management.
*   **[Headscale](roles/headscale/README.md)** 🛠️
    Installs and configures Headscale, a self-hosted implementation of Tailscale, including setting up necessary directories, configuration files, and starting the Headscale container.
*   **[Traefik](roles/traefik/README.md)** 🚦
    Installs and configures Traefik as a reverse proxy and load balancer, managing routing for services and providing SSL termination with Let's Encrypt.
*   **[Authentik](roles/authentik/README.md)** 🛂
    Installs and configures Authentik in a docker container and provides the necessary configuration for Traefik to include the service into the reverse proxy.

## Usage Examples
You find more complete usage examples under `examples/`.

### Initial Setup
This assumes a vanilla Ubuntu instance on which the user `ansible` will be set up and will get access with an SSH key that you must provide with the `public_key_path` argument.

The playbook could look as follows:
```yaml
# prepare.yml
---
- name: Initialize Ansible user
  hosts: all
  become: yes
  vars:
    public_key: "{{ lookup('file', public_key_path) }}"
    new_ansible_user: ansible # Set your desired username here
  tasks:
    - name: Include the example role
      become: yes
      ansible.builtin.import_role:
        name: t4d.WebServerSetup.init_ansible
```

To run it:
`ansible-playbook prepare.yml -e "public_key_path=/path/to/your/public_key.pub" -e "ansible_user=ubuntu" -e "ansible_ssh_private_key_file=/path/to/your/ssh/file" -e "public_key_path=/path/to/the/used/sshkey" -i your_inventory_file`

### Django Webapplication
It is recommended to run the Initial Setup before configuring the server.

#### Server configuration
This playbook will completely set up the Web application. For this to work you must:
*   Set the Ansible user and SSH key in *your_inventory_file*;
*   Have the following variables set in a local `vault.yml` file: `vault_db_password` (the password for the PostgreSQL database), `vault_infomaniak_DNS_key` (an Infomaniak token to perform DNS challenges), and `vault_repository_token` (an HTTP token that allows read access to your Django web app);
*   Have your Django application set up to use `python-decouple` and take the following variables from the `.env` file: `DATABASE_NAME`, `DATABASE_USER`, `DATABASE_PASSWORD`, `SECRET_KEY`, `ALLOWED_HOSTS`, `STATIC_ROOT`, `MEDIA_ROOT`.

The setup playbook can then look as follows:
```yaml
# setup.yml
---
- name: Configure Django Web App
  hosts: all # or target a specific host
  vars_files:
    - vault.yml # Include the vault file
  vars:
    app_name: myDjangoApp # Name of your Django application
    db_name: my_django_db # Name of the PostgreSQL database
    db_user: my_django_user # PostgreSQL user for the database
    # certbot & nginx specific setup
    server_name: myAwesomeSite.com # Your server's domain name or IP
    domain_name: myAwesomeSite.com # Domain name for the SSL certificate
    certbot_email: some@e.mail # Email for Certbot notifications
    # you likely don't want to change this
    cert_path: /etc/letsencrypt/live/{{ domain_name | lower }}/fullchain.pem
    key_path: /etc/letsencrypt/live/{{ domain_name | lower }}/privkey.pem
    # to get the web app
    git_remote: gitlab.com # or github.com or whatever
    git_repository_path: "<user>/<myapp>.git" # Git repository URL
    git_repository_branch: main # branch or tag
    # gunicorn setup
    gunicorn_workers: 3 # Number of Gunicorn workers
  roles:
    - t4d.WebServerSetup.basic_config # Added the basic_config role here
    - t4d.WebServerSetup.hardenServer
    - t4d.WebServerSetup.postgresqlSetup
    - t4d.WebServerSetup.certbot
    - t4d.WebServerSetup.nginxWebServer
    - t4d.WebServerSetup.installWebApp
    - t4d.WebServerSetup.celerySetup
    - t4d.WebServerSetup.gunicornSetup
```

To run it:
`ansible-playbook setup.yml --ask-vault-pass -i your_inventory_file`

#### Update Webapplication
This playbook assumes a fully configured web server on which it updates the Django application.
```yaml
---
- name: Update Django Web Application
  hosts: all # or target the specific host
  vars_files:
    - vault.yml # Include the vault file
  vars:
    app_name: myDjangoApp # Name of your Django application
    # to get the web app
    git_remote: gitlab.com # or github.com or whatever
    git_repository_path: "<user>/<myapp>.git" # Git repository URL
    git_repository_branch: main # branch or tag
  roles:
    - t4d.WebServerSetup.updateWebApp
```
