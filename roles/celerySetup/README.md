# Ansible Role: Celery Setup

[![build](https://img.shields.io/github/actions/workflow/status/t4d-gmbh/WebServerSetup/molecule-celerySetup.yml?label=build)](https://github.com/t4d-gmbh/WebServerSetup/actions/workflows/molecule-celerySetup.yml)

This Ansible role sets up Celery as a task queue for your web application.
It installs necessary packages, configures RabbitMQ as a message broker, and sets up Celery to run as a systemd service.

## Requirements

- Ansible 2.9 or higher
- Docker (for RabbitMQ)
- Access to a server with `systemd`
- A web application compatible with Celery

## Role Variables

- `rabbitmq_version`: The version of RabbitMQ to pull (default: latest).
- `rabbitmq_log_max_size`: Maximum size of the RabbitMQ container log file before rotation (default: `"10m"`). Accepts Docker log size notation (`10m`, `100m`, etc.).
- `rabbitmq_log_max_file`: Number of rotated RabbitMQ container log files to retain (default: `"3"`).
- `rabbitmq_memory_limit`: Docker memory limit for the RabbitMQ container (default: `"512m"`). Prevents the container from competing unboundedly for RAM.
- `rabbitmq_config_dir`: Host directory where `rabbitmq.conf` is rendered before being mounted into the container (default: `/etc/rabbitmq`).
- `rabbitmq_vm_memory_high_watermark`: RabbitMQ memory high-watermark as a fraction of the container's memory limit (default: `"0.1"`). Written to `rabbitmq.conf`; RabbitMQ blocks publishers above this threshold.
- `rabbitmq_vm_memory_high_watermark_paging_ratio`: Fraction of the high-watermark at which RabbitMQ starts paging messages to disk (default: `"0.8"`). Written to `rabbitmq.conf`.
- `celery_worker_name`: The name of the Celery worker (default: "worker1").
- `celery_time_limit`: Time limit for Celery tasks (default: 300 seconds).
- `celery_concurrency`: Number of concurrent Celery tasks (default: 8).
- `celery_log_level`: Logging level for Celery (default: "DEBUG").
- `celery_log_file`: Path for Celery log files (default: "/var/log/celery/%n%I.log").
- `celery_pid_file`: Path for Celery PID files (default: "/var/run/celery/%n.pid").
- `celery_user`: User under which Celery will run (default: "django").
- `celery_group`: Group under which Celery will run (default: "www-data").
- `django_project_subdir`: Subdirectory within `/opt/<app_name>` that contains `manage.py`. Defaults to `"."` (repository root). Set to `"{{ app_name | lower }}"` for the legacy layout.

#### Migration Note — `django_project_subdir`

`WorkingDirectory` in the Celery service file and `CELERYD_CHDIR` in the Celery config are now controlled by `django_project_subdir` (default `"."`). **Existing deployments** where `manage.py` lives inside a `<app_name>/` subdirectory must set:

```yaml
django_project_subdir: "{{ app_name | lower }}"
```

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
4. **Deploy RabbitMQ Config**: Renders `rabbitmq.conf` with memory high-watermark settings and mounts it into the container.
5. **Run RabbitMQ Container**: Starts the RabbitMQ container with memory limits and log rotation configured.
6. **Deploy Temporary Files**: Creates temporary files for Celery configuration.
7. **Ensure Configuration Directory Exists**: Ensures the `/etc/conf.d` directory exists.
8. **Ensure Celery Specific Folders Exist**: Creates necessary directories for Celery logs and runtime.
9. **Deploy logrotate config for Celery logs**: Installs a logrotate config under `/etc/logrotate.d/` to rotate `/var/log/celery/*.log` daily, retaining 14 compressed files. Uses `copytruncate` because Celery keeps log file descriptors open.
10. **Create Celery Config File**: Generates the Celery configuration file.
11. **Start and Enable Celery Service**: Starts the Celery service and enables it to run on boot.

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
