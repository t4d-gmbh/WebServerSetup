# Ansible Collection: 🌐Webserver ⚙️ Automation

This Ansible collection provides a set of roles designed to automate the setup and management of web servers, specifically tailored for deploying Django applications.
The collection includes roles for user initialization, server hardening, PostgreSQL setup, SSL certificate management, Nginx configuration, web application installation, Gunicorn setup, and application updates.

## Roles Included

1. **Init Ansible** 👤
   - Sets up a new user for Ansible automation, configures sudo access, sets up SSH keys, and disables password authentication for enhanced security.

2. **Harden Server** 🔒
   - Hardens an Ubuntu server by implementing security best practices, including configuring the Uncomplicated Firewall (UFW) and securing SSH access.

3. **PostgreSQL Setup** 🐘
   - Installs and configures PostgreSQL using Docker, ensuring that the necessary packages are installed and a PostgreSQL container is created and running.

4. **Certbot** 🔑
   - Installs and configures Certbot for obtaining and managing SSL certificates using the Infomaniak DNS plugin, ensuring automatic renewal of certificates.

5. **Nginx Web Server** 🌍
   - Installs and configures Nginx as a web server for a Django application, managing SSL certificates and serving static files.

6. **Install Web App** 📦
   - Installs and configures a Django web application, setting up the necessary environment, creating a dedicated user, and managing application secrets.

7. **Gunicorn Setup** 🚀
   - Sets up Gunicorn as a WSGI server for your web application, configuring it to run as a systemd service.

8. **Celery Setup** 🍃
   - Sets up Celery as a task queue for your web application, configuring RabbitMQ as a message broker and managing Celery as a systemd service.

9. **Update Web App** 🔄
   - Updates and manages a Django web application by fetching the latest changes from the Git repository, installing dependencies, collecting static files, and applying database migrations.

## Usage

To use this collection, include the desired roles in your playbook. Below is an example of how to use multiple roles from this collection:

```yaml
- hosts: webservers
  become: yes
  roles:
    - init_ansible
    - hardenServer
    - postgresqlSetup
    - certbot
    - nginxWebServer
    - installWebApp
    - gunicornSetup
    - celerySetup
    - updateWebApp
```

## Example

- **Initial setup**: This assumes a vanilla ubuntu instance on which the
  user `ansible` will be setup and will get access with a ssh key that you
  must provide with the `public_key_path` argument.

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

  ```bash
  ansible-playbook prepare.yml -e "public_key_path=/path/to/your/public_key.pub" -e "ansible_user=ubuntu" -e "ansible_ssh_private_key_file=/path/to/your/ssh/file" -e "public_key_path=/path/to/the/used/sshkey" -i your_inventory_file
  ```

- **Server configuration**: This playbook will completely setup the Web application.
  For this to work you must:

  - set the Ansible user and ssh key in _your_inventory_file_;
  - have the following variables set in a local `valut.yml` file:

    - `vault_db_password`: The password for the postres database to use
    - `vault_infomaniak_DNS_key`: A infomaniak token to preform DNS challenge 
    - `vault_repository_token`: A http token that allows read access to your django web app;
  - have your Django application set up to use `python-decouple` and take the following variables from the `.env` file:

    - `DATABASE_NAME`
    - `DATABASE_USER`
    - `DATABASE_PASSWORD`
    - `SECRET_KEY`
    - `ALLOWED_HOSTS`
    - `STATIC_ROOT`
    - `MEDIA_ROOT`
  
  The setup playbook can then looks as follows:

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

  ```bash
  ansible-playbook prepare.yml -e "public_key_path=/path/to/your/public_key.pub" -e "ansible_user=ubuntu" -e "ansible_ssh_private_key_file=/path/to/your/ssh/file" -e "public_key_path=/path/to/the/used/sshkey" -i your_inventory_file
  ```

- **Server configuration**: This playbook will completely setup the Web application.
  For this to work you must:

  - set the Ansible user and ssh key in _your_inventory_file_;
  - have the following variables set in a local `valut.yml` file:

    - `vault_db_password`: The password for the postres database to use
    - `vault_infomaniak_DNS_key`: A infomaniak token to preform DNS challenge 
    - `vault_repository_token`: A http token that allows read access to your django web app;
  - have your Django application set up to use `python-decouple` and take the following variables from the `.env` file:

    - `DATABASE_NAME`
    - `DATABASE_USER`
    - `DATABASE_PASSWORD`
    - `SECRET_KEY`
    - `ALLOWED_HOSTS`
    - `STATIC_ROOT`
    - `MEDIA_ROOT`
  
  The setup playbook can then looks as follows:

  ```yaml
  # setup.yml
  ---
  - name: Configure Django Web App
    hosts: all  # or target a specific host
    vars_files:
      - vault.yml  # Include the vault file
    vars:
      app_name: myDjangoApp           # Name of your Django application
      db_name: my_django_db           # Name of the PostgreSQL database
      db_user: my_django_user         # PostgreSQL user for the database
      # certbot & nginx specific setup
      server_name: myAwesomeSite.com  # Your server's domain name or IP
      domain_name: myAwesomeSite.com  # Domain name for the SSL certificate
      certbot_email: some@e.mail      # Email for Certbot notifications
      # you likely don't want to change this
      cert_path: /etc/letsencrypt/live/{{ domain_name | lower }}/fullchain.pem
      key_path: /etc/letsencrypt/live/{{ domain_name | lower }}/privkey.pem
      # to get the web app
      git_remote: gitlab.com # or github.com or whatever
      git_repository_path: "<user>/<myapp>.git"  # Git repository URL
      git_repository_branch: main     # branch or tag
      # gunicorn setup
      gunicorn_workers: 3             # Number of Gunicorn workers
    roles:
      - t4d.WebServerSetup.hardenServer
      - t4d.WebServerSetup.postgresqlSetup
      - t4d.WebServerSetup.certbot
      - t4d.WebServerSetup.nginxWebServer
      - t4d.WebServerSetup.installWebApp
      - t4d.WebServerSetup.celerySetup
      - t4d.WebServerSetup.gunicornSetup
  ```

  To run it:
  ```bash
  ansible-playbook setup.yml --ask-vault-pass -i your_inventory_file
  ```

- **Update Webapp**: This playbook assumes a fully configured web server on which it updates the django application.

  ```yaml
  ---
  - name: Update Django Web Application
    hosts: all  # or target the specific host
    vars_files:
      - vault.yml  # Include the vault file
    vars:
      app_name: myDjangoApp           # Name of your Django application
      # to get the web app
      git_remote: gitlab.com          # or github.com or whatever
      git_repository_path: "<user>/<myapp>.git"  # Git repository URL
      git_repository_branch: main     # branch or tag
    roles:
      - t4d.WebServerSetup.updateWebApp
  ```

- **Setting up a Headscale control plane**

```yaml
---
- name: Deploy Headscale behind Traefik
  hosts: all
  become: true
  vars_files:
    - vault.yml  # Load variables from the vault
  vars:
    dancer_user: "dancer"  # VARIABLE: User for docker
    server_url: "https://example.com"  # VARIABLE: Server URL
    email: "example@email.com"  # VARIABLE: Email for ACME
    dns_provider: "infomaniak"  # VARIABLE: DNS provider
    prefixes_v4: "100.64.0.0/10"  # VARIABLE: Default IPv4 prefix
    prefixes_v6: "fd7a:115c:a1e0::/48"  # VARIABLE: Default IPv6 prefix
    nameservers:
      - "1.1.1.1"  # VARIABLE: Default nameserver
      - "9.9.9.9"  # VARIABLE: Default nameserver
    additional_nameservers: []  # VARIABLE: Allow to set further nameservers

  tasks:

  roles:
    - t4d.WebServerSetup.headscale
    - t4d.WebServerSetup.traefik
```
