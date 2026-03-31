# Ansible Role: Build Frontend

[![build](https://img.shields.io/github/actions/workflow/status/t4d-gmbh/WebServerSetup/molecule-buildFrontend.yml?label=build)](https://github.com/t4d-gmbh/WebServerSetup/actions/workflows/molecule-buildFrontend.yml)

This role installs Node.js and builds frontend assets (Vite, Tailwind CSS, etc.) for a Django web application. It is designed to run after the application code has been cloned (by `installWebApp` or `updateWebApp`) and produces compiled, hashed static assets that Django's `collectstatic` then gathers into `STATIC_ROOT`.

## Requirements

- Ansible 2.9 or higher
- A Django application with a `package.json` and an npm build script at the repository root
- The application must already be cloned to the target server (via the `installWebApp` or `updateWebApp` role)
- A `django` user must exist on the target server (created by the `installWebApp` role)
- A Python virtualenv with Django installed at `/opt/<app_name>/venv/` (created by the `installWebApp` role)

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `app_name` | *(required)* | Name of the Django application. Used to derive paths. |
| `node_major_version` | `22` | Node.js major version to install from NodeSource. |
| `frontend_app_path` | `"/opt/{{ app_name }}"` | Path where `package.json` lives. |
| `frontend_build_command` | `"npm run build"` | The command to run for the production build. Can be changed to `yarn build`, `pnpm build`, etc. |
| `frontend_clean_install` | `true` | If `true`, uses `npm ci` (clean install from lockfile). If `false`, uses `npm install`. |
| `django_project_dir` | `"{{ app_name \| lower }}"` | The Django project subdirectory (where `manage.py` lives), relative to `frontend_app_path`. |
| `frontend_run_collectstatic` | `true` | Whether to run `manage.py collectstatic` after the build. |

## Dependencies

This role depends on the following roles being run beforehand:

- `t4d.WebServerSetup.installWebApp` (or `t4d.WebServerSetup.updateWebApp`) -- to clone the application code and set up the Django user, virtualenv, and `.env` file.

## Tasks Overview

1. **Install Node.js prerequisites**: Installs `ca-certificates`, `curl`, and `gnupg` via apt.
2. **Add NodeSource repository**: Adds the NodeSource GPG key and apt repository for the specified Node.js major version.
3. **Install Node.js**: Installs Node.js from the NodeSource repository.
4. **Install ACL package**: Required for Ansible `become_user` with non-root users.
5. **Install frontend dependencies**: Runs `npm ci` (or `npm install`) in the application directory as the `django` user.
6. **Build frontend assets**: Runs the configured build command (default: `npm run build`) to compile CSS/JS assets.
7. **Collect static files**: Runs Django's `collectstatic` to gather the built assets into `STATIC_ROOT`.

## Example Playbook

### Initial Setup

```yaml
- hosts: webservers
  become: yes
  roles:
    - t4d.WebServerSetup.installWebApp
    - t4d.WebServerSetup.buildFrontend
    - t4d.WebServerSetup.gunicornSetup
```

### Application Update

```yaml
- hosts: webservers
  become: yes
  roles:
    - t4d.WebServerSetup.updateWebApp
    - t4d.WebServerSetup.buildFrontend
```

### Custom Build Configuration

```yaml
- hosts: webservers
  become: yes
  vars:
    node_major_version: 20
    frontend_build_command: "npm run build:production"
    frontend_clean_install: false
  roles:
    - t4d.WebServerSetup.buildFrontend
```

## Notes

- **Vite integration**: For Django projects using `django-vite`, ensure `VITE_DEV_MODE=False` is set in the application's `.env` file for production. The `installWebApp` role's `.env` template includes this variable when `vite_dev_mode` is set.
- **Node.js versions**: The role installs Node.js from NodeSource. Only LTS versions (18, 20, 22) are recommended for production.
- **Build output**: The build command is expected to place output files within Django's `STATICFILES_DIRS` so that `collectstatic` can find them. For Vite, this is typically configured via `build.outDir` in `vite.config.js`.
- **npm ci vs npm install**: `npm ci` is recommended for deployments -- it installs from the lockfile for reproducible builds and is faster. Use `npm install` only if you don't have a `package-lock.json`.
- **Disk space**: The `node_modules/` directory remains on disk after the build (~100-300MB depending on the project). If disk space is a concern, add a cleanup task after the build.

## License

This role is licensed under the GNU GPLv3 License.

## Author Information

This role was created in 2025 by Jonas I Liechti @ T4D.ch.
