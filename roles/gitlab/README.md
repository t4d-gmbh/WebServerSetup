# Ansible Role: GitLab

[![build](https://img.shields.io/github/actions/workflow/status/t4d-gmbh/WebServerSetup/molecule-gitlab.yml?label=build)](https://github.com/t4d-gmbh/WebServerSetup/actions/workflows/molecule-gitlab.yml)

Installs **GitLab CE** from the official [omnibus apt repository](https://about.gitlab.com/install/) with a fully managed `/etc/gitlab/gitlab.rb`. The role is designed for hosts that are **not publicly reachable** (e.g. only accessible from inside a NetBird network): TLS certificates are obtained with the [certbot](../certbot/README.md) role via the **DNS-01 challenge**, and omnibus' built-in Let's Encrypt (HTTP-01) is disabled.

Apart from the certificate wiring the deployment follows the official installation procedure exactly: GitLab's own package, its own bundled nginx/PostgreSQL/Redis, and `gitlab-ctl reconfigure`.

## Requirements

- **Ubuntu 22.04 (jammy) or 24.04 (noble)** — GitLab publishes omnibus repositories only for LTS releases. The role asserts this and fails early.
- At least **4 GB RAM** (8 GB recommended), 10+ GB free disk under `/var/opt/gitlab`.
- A DNS A record for `gitlab_host` (may point at a private/VPN IP — DNS-01 does not require the host to be reachable).
- The [certbot](../certbot/README.md) role must have issued the certificate for `gitlab_host` **before** this role runs (see [examples/GitLab.md](../../examples/GitLab.md)).

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `gitlab.version` | `19.3.1-ce.0` | Exact omnibus package version to pin. |
| `gitlab_repo_codename` | `noble` | Repository suite (`jammy` or `noble`). |
| `gitlab_host` | from `server_url` | Hostname used for `external_url` and certificate lookup. |
| `gitlab_external_url` | `https://{{ gitlab_host }}` | Omnibus `external_url`. |
| `gitlab_ssl_certificate` | `/etc/letsencrypt/live/{{ gitlab_host }}/fullchain.pem` | Certificate used by omnibus nginx. |
| `gitlab_ssl_certificate_key` | `/etc/letsencrypt/live/{{ gitlab_host }}/privkey.pem` | Matching private key. |
| `gitlab_email_from` | `""` | Optional `gitlab_rails['gitlab_email_from']`. |
| `gitlab_skip_auto_reconfigure` | `true` | Keep `/etc/gitlab/skip-auto-reconfigure`, which suppresses the reconfigure on package **upgrades** (a first install always reconfigures from the postinst — the role templates `gitlab.rb` beforehand so that run uses the right config). |
| `gitlab_reconfigure` | `true` | Whether the handler actually runs `gitlab-ctl reconfigure`. CI sets this to `false`. |
| `vault_gitlab_initial_root_password` | — | Optional vault variable; applied on the **first** reconfigure only. When unset, GitLab writes `/etc/gitlab/initial_root_password` (valid for 24 h). |

## Dependencies

None at role level, but a correct deployment needs the **certbot role beforehand**:

```yaml
  vars:
    certbot_renew_hook: "gitlab-ctl restart nginx"
  roles:
    - t4d.WebServerSetup.certbot   # DNS-01 cert for gitlab_host
    - t4d.WebServerSetup.gitlab
```

`certbot_renew_hook: "gitlab-ctl restart nginx"` is required because omnibus nginx is managed by runit, not systemd — a plain `systemctl reload nginx` would fail on renewal.

## How TLS works here

1. `certbot` obtains `https://gitlab_host`'s certificate through the Infomaniak DNS-01 challenge — works even though the host only answers on a VPN interface.
2. `letsencrypt['enable'] = false` stops omnibus from attempting its own ACME validation on every reconfigure (it would always fail on a VPN-only host).
3. `nginx['ssl_certificate(_key)']` point at the certbot-managed symlinks under `/etc/letsencrypt/live/`, which certbot renews in place.
4. The certbot renewal cron (see [certbot role](../certbot/README.md)) restarts omnibus nginx via the deploy hook after each renewal.

## Example Playbook

See [examples/GitLab.md](../../examples/GitLab.md) for a complete NetBird + certbot + GitLab deployment.

## CI Notes

The molecule suite installs the real package (proving repo, pin and keyring). A first install always reconfigures from the package postinst and cannot be skipped, so `prepare.yml` generates a self-signed stand-in certificate at the certbot live paths — letting the real chef converge run inside the container. The handler-driven second reconfigure is gated off (`gitlab_reconfigure: false`), and the certificate assertion is tagged `molecule-notest`.

## License

Apache-2.0 (see repository root)

## Author

t4d GmbH
