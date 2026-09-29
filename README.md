# Ansible Scripts (Demo)

A small demo Ansible project showing workstation-provisioning patterns: cross-platform package installs (Homebrew on macOS, apt on Linux), dev tool setup, and dotfile symlinking.

This is a portfolio/doodling project put together for GitHub and my resume to demonstrate familiarity with Ansible — it is **not** intended for production use as-is (no vault secrets are configured, and the example inventory is a placeholder).

## Structure

- `site.yml` — main playbook, applies all roles to all hosts
- `inventory/hosts.yml` — example inventory (`mac` = localhost, `linux` = placeholder remote group)
- `group_vars/` — variables per group (`all`, `mac`, `linux`)
- `roles/common` — updates package cache, installs base CLI tools
- `roles/dev_tools` — installs language runtimes, Docker, AWS CLI, and (on macOS) GUI apps via Homebrew Cask
- `roles/dotfiles` — clones a dotfiles repo and symlinks selected files into `$HOME`

## Usage

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml -l mac
```

To try it against remote Linux hosts, add them under `linux:` in `inventory/hosts.yml` and run:

```bash
ansible-playbook site.yml -l linux --ask-become-pass
```

## Notes

- Package lists and the dotfiles repo URL in `group_vars/` are placeholders — edit them to match real preferences before running.
- No secrets are stored here; `.gitignore` excludes common vault/secret filenames if you extend this with `ansible-vault`.
