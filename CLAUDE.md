# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ansible playbook that provisions a Pop!_OS (or Debian/Ubuntu) desktop workstation with development tools, productivity apps, and GNOME/Cosmic desktop customization. Designed to run once after a fresh OS install.

## Commands

```bash
# Run full playbook (prompts for sudo password)
ansible-playbook local.yml

# Run with specific tags
ansible-playbook local.yml --tags steam
ansible-playbook local.yml --tags all

# Dry run / check mode
ansible-playbook local.yml --check

# Syntax check
ansible-playbook local.yml --syntax-check

# Discover GNOME dconf changes
dconf dump / > before.txt
# make changes in settings UI
dconf dump / > after.txt
diff before.txt after.txt
```

## Architecture

- **`local.yml`** — Main playbook. Runs against localhost with `become: true`. Pre-tasks validate: not root, supported distro (Debian/Ubuntu/Pop!_OS), desktop environment (GNOME/Unity/Cosmic), sudo access.
- **`group_vars/all.yml`** — Shared variables (teleport version, VS Code extensions, Chrome startup pages/extensions).
- **`ansible.cfg`** — Sets inventory to `inventory/localhost.ini`, enables `become_ask_pass`.
- **`roles/`** — Each role has only `tasks/main.yml` (no defaults, handlers, or meta). Roles are either always-run (listed directly) or tag-gated (declared with `role:` + `tags:`).
- **`{{ actual_user }}`** — The non-root user running the playbook, set via `lookup('env', 'USER')`. Used across roles for home directory paths and file ownership.

## Conventions

- Roles use the `package` module for portability, except where `apt`-specific features are needed (e.g., `deb:` installs, `update_cache`).
- Autostart entries are created via `.desktop` files in `/home/{{ actual_user }}/.config/autostart/`.
- GNOME settings are applied via `community.general.dconf` with `become_user: "{{ actual_user }}"`.
- Versions for external tools (Teleport, Brother scanner driver, NAPS2) are hardcoded in task files or `group_vars/all.yml`.
- Most roles in `local.yml` are commented out; uncomment as needed for a given setup run.

## Available Tags

`customizing`, `slides`, `suspend`, `brother-scanner`, `steam`, `vpn`, `syncthing`
