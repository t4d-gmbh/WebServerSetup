<!--
SPDX-FileCopyrightText: 2026 Jonas I. Liechti <j-i-l@t4d.ch>

SPDX-License-Identifier: GPL-3.0-or-later
-->

# Ansible Role: GitLab

[![build](https://img.shields.io/github/actions/workflow/status/t4d-gmbh/WebServerSetup/molecule-gitlab.yml?label=build)](https://github.com/t4d-gmbh/WebServerSetup/actions/workflows/molecule-gitlab.yml)

Installs **GitLab CE** from the official [omnibus apt repository](https://about.gitlab.com/install/) with a fully managed `/etc/gitlab/gitlab.rb`. The role is designed for hosts that are **not publicly reachable** (e.g. only accessible from inside a NetBird network): TLS certificates are obtained with the [certbot](../certbot/README.md) role via the **DNS-01 challenge**, and omnibus' built-in Let's Encrypt (HTTP-01) is disabled.

Apart from the certificate wiring the deployment follows the official installation procedure exactly: GitLab's own package, its own bundled nginx/PostgreSQL/Redis, and `gitlab-ctl reconfigure`.

## Requirements

- **Ubuntu 22.04 (jammy), 24.04 (noble), or 26.04 (resolute)** — GitLab publishes omnibus repositories for these releases only (no interim releases). The role asserts this and fails early.
- At least **4 GB RAM plus swap** (8 GB recommended), 10+ GB free disk under `/var/opt/gitlab`. The first reconfigure boots the full Rails environment; on ≤4 GB *without* swap the final `gitlab-rake cache:clear` can thrash until chef's 3600 s timeout kills the postinst. Provision swap first (e.g. the [basic_config](../basic_config/README.md) role with `swapfile_size: "4G"`) or set `gitlab_rake_cache_clear: false` — see [Low-memory tuning](#low-memory-tuning).
- A DNS A record for `gitlab_host` (may point at a private/VPN IP — DNS-01 does not require the host to be reachable).
- The [certbot](../certbot/README.md) role must have issued the certificate for `gitlab_host` **before** this role runs (see [examples/GitLab.md](../../examples/GitLab.md)).

## Role Variables

| Variable | Default | Description |
|---|---|---|
| `gitlab.version` | `19.3.1-ce.0` | Exact omnibus package version to pin — default, override per deployment. |
| `gitlab_repo_codename` | `{{ ansible_distribution_release }}` | Repository suite; auto-matches the host (jammy/noble/resolute). |
| `gitlab_host` | from `server_url` | Hostname used for `external_url` and certificate lookup. |
| `gitlab_external_url` | `https://{{ gitlab_host }}` | Omnibus `external_url`. |
| `gitlab_ssl_certificate` | `/etc/letsencrypt/live/{{ gitlab_host }}/fullchain.pem` | Certificate used by omnibus nginx. |
| `gitlab_ssl_certificate_key` | `/etc/letsencrypt/live/{{ gitlab_host }}/privkey.pem` | Matching private key. |
| `gitlab_email_from` | `""` | Optional `gitlab_rails['gitlab_email_from']`. |
| `gitlab_rake_cache_clear` | `true` | When `false`, emits `gitlab_rails['rake_cache_clear'] = false` — skips the memory-heavy Rails-boot cache clear at the end of reconfigure. |
| `gitlab_prometheus_monitoring` | `true` | When `false`, emits `prometheus_monitoring['enable'] = false` — disables the bundled monitoring stack. See [Low-memory tuning](#low-memory-tuning). |
| `gitlab_puma_worker_processes` | `""` | When set, emits `puma['worker_processes']` (unquoted int). |
| `gitlab_sidekiq_max_concurrency` | `""` | When set, emits `sidekiq['max_concurrency']` (unquoted int). |
| `gitlab_postgresql_shared_buffers` | `""` | When set, emits `postgresql['shared_buffers']` (quoted, e.g. `"256MB"`). |
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
3. `gitlab_rails['nginx']['ssl_certificate(_key)']` point at the certbot-managed symlinks under `/etc/letsencrypt/live/`, which certbot renews in place. (Since GitLab 19.2 these rails-specific settings live under `gitlab_rails['nginx']`; the legacy `nginx['ssl_certificate*']` keys are deprecated one-way shims and are *not* used by this role.)
4. The certbot renewal cron (see [certbot role](../certbot/README.md)) restarts omnibus nginx via the deploy hook after each renewal.

## Low-memory tuning

Omnibus defaults (full monitoring stack, sidekiq concurrency 20+) target bigger machines; on a 4 GB VPS they cause constant swap thrashing — pages evicted and faulted back on every request (symptoms: multi-second TTFB even over loopback, high `si`/`so` in `vmstat`, disk at ~100% util with idle CPU). All knobs below emit nothing unless set, so defaults render the same `gitlab.rb` as before.

Recommended profile for a 4 vCPU / 4 GB + swap host with a handful of users (per GitLab's [instance limits](https://docs.gitlab.com/administration/instance_limits/) guidance):

```yaml
gitlab_rake_cache_clear: false          # keeps reconfigure itself from thrashing
gitlab_prometheus_monitoring: false     # drops prometheus/grafana/alertmanager/exporters (~0.5-1 GB)
gitlab_puma_worker_processes: 2         # each worker is ~600 MB
gitlab_sidekiq_max_concurrency: 5       # 10 is the documented 4 GB value; 5 for very low traffic
gitlab_postgresql_shared_buffers: "256MB"
```

Verify after rollout: `vmstat 1` should show `si`/`so` ≈ 0, swap usage drops, and `curl -k https://localhost/help` TTFB back in the 1–2 s range.

## Example Playbook

See [examples/GitLab.md](../../examples/GitLab.md) for a complete NetBird + certbot + GitLab deployment.

## Known upstream issues

- **Ubuntu 26.04 (resolute) installs are currently impossible via the resolute suite** (observed 2026-10-05): GitLab's resolute `Packages` index lists its own per-distro artifacts, but the shared pool URLs (`/ubuntu/pool/main/g/gitlab-ce/gitlab-ce_<version>_amd64.deb`) resolve to **noble's** files for every published version (19.3.1–19.4.1 verified). apt therefore always aborts with `File has unexpected size … Mirror sync in progress?`. Re-check with:
  `curl -s https://packages.gitlab.com/gitlab/gitlab-ce/ubuntu/dists/resolute/main/binary-amd64/Packages | grep -E "Version|SHA256|Size"` vs the artifact id in the redirect of `curl -sIL https://packages.gitlab.com/gitlab/gitlab-ce/ubuntu/pool/main/g/gitlab-ce/gitlab-ce_19.4.1-ce.0_amd64.deb`.

  **Workaround on 26.04 hosts:** set `gitlab_repo_codename: "noble"`. The noble deb's only dependencies (`openssh-server`, `perl`) are satisfied on 26.04, omnibus is self-contained, and GitLab 19.3.x officially supports 26.04 — so chef's platform check passes and the noble artifact (which is what the pool serves anyway) installs cleanly. The molecule `instance2604` platform validates exactly this path; drop the override once GitLab republishes resolute.

## Troubleshooting

- **`Install GitLab CE` fails**: the package postinst runs the first `gitlab-ctl reconfigure` and swallows its output; chef logs to `/var/log/gitlab/reconfigure/<timestamp>.log`. The role's rescue surfaces the newest log's tail in the failure message. After fixing the root cause:
  ```bash
  sudo gitlab-ctl reconfigure          # iterate until clean (full console output)
  sudo dpkg --configure gitlab-ce      # finalize the half-configured package
  ```
  then rerun the playbook. Common cause on small hosts: memory thrash during `gitlab-rake cache:clear` hitting chef's 3600 s timeout — see [Low-memory tuning](#low-memory-tuning).
- **`File has unexpected size … Mirror sync in progress`**: on 26.04 this is the upstream resolute publication bug — see [Known upstream issues](#known-upstream-issues) and the `gitlab_repo_codename: "noble"` workaround.

## gitlab-secrets.json — state, not config

`/etc/gitlab/gitlab-secrets.json` is generated by the **first** reconfigure (OTP/2FA keys, ActiveRecord encryption keys, registry/Pages secrets) and never rewritten afterwards. The role deliberately does not manage it — and unlike `gitlab.rb`, a lost secrets file is **not** recovered by rerunning Ansible: the next reconfigure silently generates *fresh* keys, permanently breaking decryption of existing 2FA tokens and encrypted data.

Protection (outside the role's scope by design):

- The omnibus package installs a nightly `gitlab-ctl backup-etc` cron snapshotting `/etc/gitlab` **including** `gitlab-secrets.json` — verify with `ls /etc/gitlab/config_backup/`.
- Those snapshots live on the same disk as the data they protect: copy `gitlab-secrets.json` (and `gitlab.rb`) off-host as part of your backup routine.

## CI Notes

The molecule suite installs the real package (proving repo, pin and keyring). A first install always reconfigures from the package postinst and cannot be skipped, so `prepare.yml` generates a self-signed stand-in certificate at the certbot live paths — letting the real chef converge run inside the container. The handler-driven second reconfigure is gated off (`gitlab_reconfigure: false`), and the certificate assertion is tagged `molecule-notest`.

## License

This role is licensed under the GNU GPLv3 License.

## Author

t4d GmbH
