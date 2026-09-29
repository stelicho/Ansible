# Ansible Scripts (Demo)

A portfolio Ansible project demonstrating a range of provisioning and orchestration patterns: cross-platform workstation setup, a multi-tier web app deploy with a rolling/health-checked rollout, Docker container orchestration, CIS-lite security hardening, and a Prometheus + Grafana monitoring stack whose scrape targets are generated from live inventory.

This is a **portfolio/demo project** put together for GitHub and my resume to show familiarity with Ansible at a level beyond basic package installs — it is not intended to be pointed at real production infrastructure as-is (the inventory is all placeholder hosts).

## Playbooks

| Playbook | What it shows | Target group(s) |
| --- | --- | --- |
| `site.yml` | Base workstation provisioning (packages, dev tools, dotfiles) | `mac`, `linux` |
| `webapp.yml` | Multi-tier deploy (Postgres → app → nginx), `serial` rolling rollout with a health-check gate, vaulted DB credentials | `dbservers`, `appservers`, `webservers` |
| `containers.yml` | Docker Engine install + a templated `docker-compose.yml` stack deployed via `community.docker`, with a rolling-restart handler | `dockerhosts` |
| `hardening.yml` | CIS-lite baseline: SSH lockdown with a `block`/`rescue` safety net, UFW default-deny, fail2ban, unattended upgrades, tag-based subsetting | `hardened` (group-of-groups) |
| `monitoring.yml` | node_exporter fleet-wide, Prometheus with a scrape config templated straight from inventory groups, Grafana with a provisioned datasource and vaulted admin password | `webservers`/`appservers`/`dbservers`/`dockerhosts`/`monitoring` |

## Structure

- `inventory/hosts.yml` — placeholder hosts for every tier, plus a `hardened` group that composes the other groups so `hardening.yml` doesn't need to repeat host lists
- `group_vars/` — variables per group; `dbservers/` and `monitoring/` use the `vars.yml` + `vault.yml` split (see **Secrets** below)
- `roles/common`, `roles/dev_tools`, `roles/dotfiles` — workstation provisioning (unchanged from the original demo)
- `roles/postgres`, `roles/app`, `roles/nginx` — the three tiers behind `webapp.yml`
- `roles/docker`, `roles/containers_app` — Docker Engine + a compose-based stack
- `roles/hardening` — SSH/firewall/fail2ban/updates, split into per-concern task files with tags
- `roles/node_exporter`, `roles/prometheus`, `roles/grafana` — the monitoring stack

## Usage

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml -l mac
```

Every other playbook targets placeholder Linux hosts — uncomment/add real hosts under the relevant group in `inventory/hosts.yml`, then:

```bash
ansible-playbook webapp.yml     --ask-vault-pass --ask-become-pass
ansible-playbook containers.yml --ask-become-pass
ansible-playbook hardening.yml  --ask-become-pass --tags ssh,firewall   # or run the whole role
ansible-playbook monitoring.yml --ask-vault-pass --ask-become-pass
```

## Secrets (ansible-vault)

`group_vars/dbservers/` and `group_vars/monitoring/` each split into:

- `vars.yml` — plain variables, referencing `vault_*` names
- `vault.yml` — the same directory's secrets, encrypted whole-file with `ansible-vault`

This is the standard Ansible pattern for keeping secrets alongside the variables that use them, encrypted at rest, and safe to commit — `vault.yml` here **is** checked into git on purpose.

For this demo only, the vault password is `demo-vault-password-123` (never do this for real secrets — normally the password lives in a password manager, a `--vault-password-file` outside the repo, or a CI secret, never in the README):

```bash
echo -n 'demo-vault-password-123' > .vault_pass   # gitignored
ansible-vault view --vault-password-file .vault_pass group_vars/dbservers/vault.yml
ansible-playbook webapp.yml --vault-password-file .vault_pass
```

## CI / testing

- `.github/workflows/lint.yml` runs `yamllint` and `ansible-lint` (production profile) on every push/PR
- `roles/nginx/molecule/default/` is a [Molecule](https://ansible.readthedocs.io/projects/molecule/) scenario (Docker driver) that converges the role in a container and verifies nginx is active and the health endpoint responds:

  ```bash
  pip install molecule molecule-plugins[docker]
  cd roles/nginx && molecule test
  ```

## Notes

- Package lists, the dotfiles repo URL, and the app/docker image references in `group_vars/` are placeholders — swap them for real ones before running against actual hosts.
- The hardening and monitoring roles assume a Debian/Ubuntu (`apt`) target; the workstation roles (`common`, `dev_tools`, `dotfiles`) still support both macOS (Homebrew) and Debian.
- `roles/hardening/tasks/ssh.yml` runs `sshd -t` before restarting sshd and uses `block`/`rescue` to fail loudly instead of reloading a broken config — a real-world gotcha worth guarding against deliberately.
