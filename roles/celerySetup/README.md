# Ansible Role: Celery Setup

This Ansible role sets up Celery as a task queue for your web application.
It installs necessary packages, configures RabbitMQ as a message broker, and sets up Celery to run as a systemd service.

## Requirements

- Ansible 2.9 or higher
- Docker (for RabbitMQ)
- Access to a server with `systemd`
- A web application compatible with Celery

## Role Variables

- `rabbitmq_version`: The version of RabbitMQ to pull (default: latest).
- `celery_worker_name`: The name of the Celery worker (default: "worker1").
- `celery_time_limit`: Time limit for Celery tasks (default: 300 seconds).
- `celery_concurrency`: Number of concurrent Celery tasks (default: 8).
- `celery_log_level`: Logging level for Celery (default: "DEBUG").
- `celery_log_file`: Path for Celery log files (default: "/var/log/celery/%n%I.log").
- `celery_pid_file`: Path for Celery PID files (default: "/var/run/celery/%n.pid").
- `celery_user`: User under which Celery will run (default: "django").
- `celery_group`: Group under which Celery will run (default: "www-data").

## Dependencies

This role requires the `community.docker` collection for managing Docker containers.

## Installation

To use this role, add it to your Ansible playbook as follows:

```yaml
- hosts: your_target_hosts
  roles:
    - celerySetup
```

## Tasks Overview

1. **Ensure ACL Package is Installed**: Installs the ACL package to manage permissions.
2. **Create Celery systemd Service File**: Configures a systemd service for Celery.
3. **Pull RabbitMQ Docker Image**: Pulls the specified RabbitMQ Docker image.
4. **Run RabbitMQ Container**: Starts the RabbitMQ container with the necessary configurations.
5. **Deploy Temporary Files**: Creates temporary files for Celery configuration.
6. **Ensure Configuration Directory Exists**: Ensures the `/etc/conf.d` directory exists.
7. **Ensure Celery Specific Folders Exist**: Creates necessary directories for Celery logs and runtime.
8. **Create Celery Config File**: Generates the Celery configuration file.
9. **Start and Enable Celery Service**: Starts the Celery service and enables it to run on boot.
10. **Notify Handlers to Restart Celery Service**: Notifies handlers to restart the service if configuration changes.

## Handlers Overview

- **Reload systemd**: Reloads the systemd daemon to recognize new service files.
- **Restart Celery Service**: Restarts the Celery service.

## Usage

1. Define the required variables in your playbook or inventory.
2. Run the playbook to apply the role.

## Example Playbook

```yaml
- hosts: webservers
  become: yes
  vars:
    rabbitmq_version: "latest"
    celery_worker_name: "worker1"
    celery_time_limit: 300
    celery_concurrency: 8
    celery_log_level: "DEBUG"
    celery_user: "django"
    celery_group: "www-data"
  roles:
    - celerySetup
```

## License

This role is licensed under the GNU GPLv3 License.

## Author Information

This role was created in 2024 by Jonas I Liechti @ T4D.ch.
