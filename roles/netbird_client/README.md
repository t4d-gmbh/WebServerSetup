# Ansible Role: NetBird Client

[![build](https://img.shields.io/github/actions/workflow/status/t4d-gmbh/WebServerSetup/molecule-netbird_client.yml?label=build)](https://github.com/t4d-gmbh/WebServerSetup/actions/workflows/molecule-netbird_client.yml)

Installs the **NetBird client** from the official apt repository (`pkgs.netbird.io/debian`) and enrolls the host into a NetBird network with a **setup key**. Intended for servers that should only be reachable through the VPN overlay — e.g. a GitLab CE instance fronted by the [netbird](../netbird/README.md) server you self-hosted.

This is the client side; the [netbird](../netbird/README.md) role deploys the server side.

## Requirements

- A running NetBird deployment (self-hosted via the [netbird role](../netbird/README.md) or NetBird Cloud) whose **management URL is reachable from this host over the regular network** — enrollment happens before the host is inside the VPN.
- A setup key created in the dashboard under **Peers → Access Keys**.

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `netbird_client_management_url` | `""` (required) | Public URL of the management endpoint, e.g. `https://nb.example.com`. |
| `netbird_client_version` | `""` | Optional exact package version pin; `""` installs the repo's latest. |
| `netbird_client_name` | `{{ ansible_hostname }}` | Peer name shown in the dashboard. |
| `netbird_client_start_service` | `true` | Enable/start `netbird.service` and enroll. CI sets `false` (no TUN device). |
| `vault_netbird_setup_key` | — | **Required** vault variable: setup key for `netbird up`. |
| `vault_netbird_pre_shared_key` | — | Optional vault variable: PSK additionally required by the server. |

## Dependencies

None. Pairs with the [netbird](../netbird/README.md) role on the server side.

## Behavior

1. Adds the official apt repository and keyring (`/etc/apt/keyrings/netbird.asc`).
2. Installs the `netbird` package and enables the systemd service.
3. Checks `netbird status`; runs `netbird up --management-url ... --setup-key ...` only when the daemon reports `NeedsLogin`. The task is `no_log` so keys never appear in output.
4. Enrollment persists (`/etc/netbird/config.json`); the peer reconnects automatically after reboots.

Verify in the dashboard: the peer appears with a 100.64.0.0/10 address.

## Example Playbook

```yaml
- name: Join the VPN overlay
  hosts: gitlab
  become: true
  vars_files:
    - vault.yml
  vars:
    netbird_client_management_url: "https://nb.example.com"
  roles:
    - t4d.WebServerSetup.netbird_client
```

## CI Notes

The molecule suite installs the real package from the official repo and verifies the CLI; service start and enrollment are gated (`netbird_client_start_service: false` / `molecule-notest`) since CI containers have no TUN device and no reachable management server.
