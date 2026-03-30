# Ansible Role: Basic Configuration

This role manages **foundational server configurations**, primarily focusing on **swap space management** and **kernel parameter tuning** for optimal server performance. When `swapfile_size` is set, it creates and activates a swap file and tunes memory management kernel parameters. When `swapfile_size` is left as `null` (the default), the entire role is skipped.

## Requirements

- Ansible 2.9 or higher
- Access to a server with root privileges
- The `ansible.posix` collection (for `sysctl` tasks)

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `swapfile_size` | `null` | Size of the swap file, e.g. `"1G"` or `"2048M"`. When `null` or empty, no swap is configured and the role is skipped entirely. Only uppercase `G` and `M` suffixes are supported. |
| `swapfile_path` | `"/swapfile"` | Absolute path where the swap file will be created. |
| `swappiness_value` | `"10"` | Kernel `vm.swappiness` parameter (0-100). Lower values prioritize keeping data in RAM. A value of 10 is generally recommended for servers. |
| `vfs_cache_pressure_value` | `"50"` | Kernel `vm.vfs_cache_pressure` parameter. Lower values retain filesystem metadata (inode/dentry) in cache longer, improving filesystem performance. |

All variables have defaults -- none are strictly required. However, setting `swapfile_size` to a valid value (e.g. `"2G"`) is necessary for the role to perform any action.

## Dependencies

None. This role has no dependencies on other roles, but the `ansible.posix` collection must be installed.

## Tasks Overview

1. **Check if swap file exists**: Uses `stat` to check whether the file at `swapfile_path` already exists.
2. **Create swap file**: Creates the swap file with `dd` (only if it does not already exist).
3. **Set file permissions**: Sets the swap file to `0600` (root read/write only).
4. **Format as swap**: Runs `mkswap` to initialize the file as swap space (only when the file was newly created).
5. **Activate swap**: Runs `swapon` to enable the swap file. Gracefully handles the case where it is already active.
6. **Persist in fstab**: Adds an entry to `/etc/fstab` so the swap file is activated automatically on reboot.
7. **Tune vm.swappiness**: Sets the kernel swappiness parameter via `sysctl`.
8. **Tune vm.vfs_cache_pressure**: Sets the kernel vfs_cache_pressure parameter via `sysctl`.

All tasks are wrapped in a block that is skipped when `swapfile_size` is `null` or empty.

## Example Playbook

```yaml
---
- name: Apply basic server configurations
  hosts: all
  become: yes
  roles:
    - role: t4d.WebServerSetup.basic_config
      vars:
        swapfile_size: "2G"
        swappiness_value: "10"
        vfs_cache_pressure_value: "50"
```

This example creates a 2GB swap file at `/swapfile` and sets the kernel memory parameters to their recommended server values.

## Notes

- **Sizing guidance**: An amount equal to or double the system's RAM is a common starting point. Anything over 4G is generally unnecessary if swap is only used as a RAM fallback.
- **Idempotency**: The role is safe to run multiple times. Swap file creation and formatting are skipped if the file already exists, and `swapon` gracefully handles an already-active swap file.
- **Supported size formats**: Only `G` (gigabytes) and `M` (megabytes) suffixes are supported, e.g. `"2G"`, `"512M"`. Integer values only -- decimals like `"1.5G"` are not supported.

## License

This role is licensed under the GNU GPLv3 License.

## Author Information

This role was created in 2025 by Jonas I Liechti @ T4D.ch.
