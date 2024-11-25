# Ansible Collection: 🌐Webserver 🌐Automation

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

8. **Update Web App** 🔄
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
    - updateWebApp
```

## Role Variables

Each role has its own set of variables that can be defined in your playbook or inventory.
Refer to the individual role documentation for details on the required and optional variables.
**More details are available in the README files of each role.**

## Dependencies

Some roles may have dependencies on other roles or collections.
Ensure that you have the necessary collections installed, such as `community.docker` for the PostgreSQL setup role.

## Security Considerations

- Always review and customize the roles according to your specific environment and security requirements.
- Test the roles in a safe environment before deploying them to production.

## Contributing

Contributions are welcome! Please submit a pull request or open an issue for any enhancements or bug fixes.

## License

This collection is licensed under the MIT License. See the file for more details.

## Copyright

© 2024 Jonas I Liechti, t4d.ch. All rights reserved. 

