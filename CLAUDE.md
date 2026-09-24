# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An Ansible project for declaratively managing Debian/Ubuntu servers (users, firewall, packages,
Docker apps). All host-specific configuration lives in inventory/`group_vars` YAML — role code is
never edited to configure a specific server.

The repo has two layers:
- **The consuming project** (`playbooks/`, `ansible.cfg`, `requirements.yml`, `examples/`,
  `inventory/`) — orchestrates everything for a real set of servers.
- **`ansible-collection-server-management/`** — a self-contained, independently publishable
  Ansible Collection (`mishikama.server_management`, namespace `mishikama`) holding the three
  custom roles. It has its own `galaxy.yml`/README and could be `git init`'d and pushed as its
  own repo at any time (see its README for the extraction steps) — don't assume paths from the
  outer project (e.g. `../inventory`) are valid inside it.

Three roles come from public Galaxy roles (`geerlingguy.security`, `geerlingguy.firewall`,
`geerlingguy.docker`) and are installed into `roles/` — that directory is gitignored
(`roles/geerlingguy.*`) and reinstalled via `requirements.yml`, never edited or committed. Three
roles are custom, live in `ansible-collection-server-management/roles/`, and are the only role
code that should normally be changed:

| Role | FQCN | Purpose |
|---|---|---|
| `users` | `mishikama.server_management.users` | Create/remove local users, deploy SSH keys, manage sudo/sudoers |
| `packages` | `mishikama.server_management.packages` | Install packages, `apt dist-upgrade`, autoremove, conditional reboot |
| `docker_apps` | `mishikama.server_management.docker_apps` | Sync docker-compose stacks to a server and manage started/stopped/absent state |

## Local collection resolution

The custom collection does not need installing — `ansible.cfg` sets
`collections_path = .collections:~/.ansible/collections`, and `.collections/ansible_collections/mishikama/server_management`
is a symlink to `ansible-collection-server-management/`. Edits under
`ansible-collection-server-management/roles/*` are immediately live in the playbooks with no
build/install step.

## Commands

```bash
# Install the public geerlingguy roles + community/posix collections (required once, and after bumping requirements.yml)
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml

# Set up a real inventory (examples/ is a demo, not meant to be edited in place)
cp -r examples/inventory inventory

# Full run (packages -> security -> firewall -> users -> docker -> apps, in that order)
ansible-playbook playbooks/site.yml

# Run a single area
ansible-playbook playbooks/packages.yml
ansible-playbook playbooks/security.yml
ansible-playbook playbooks/firewall.yml
ansible-playbook playbooks/users.yml
ansible-playbook playbooks/docker.yml
ansible-playbook playbooks/apps.yml

# Dry run / diff before applying
ansible-playbook playbooks/site.yml --check --diff

# Limit to one host
ansible-playbook playbooks/site.yml --limit srv1.example.com

# Syntax-check a playbook
ansible-playbook playbooks/site.yml --syntax-check
```

There is no test suite, linter config, or CI in this repo — validate changes with
`--syntax-check` and `--check --diff` against the example inventory (`ansible.cfg` already points
`inventory` at `examples/inventory/hosts.yml` by default), since there's no live server to apply
against from this environment.

## Key conventions

- **Inventory groups**: `debian_servers` = every managed server; `docker_hosts` = the subset that
  also gets Docker + compose apps. Each playbook targets one of these two groups.
- **`examples/` vs `inventory/`/`files/`**: `examples/` is the checked-in demo (inventory +
  sample docker-compose stacks); `inventory/` and `files/` are the real, gitignored, per-deployment
  copies a user creates locally. Never commit real hosts, keys, or app files — only touch
  `examples/`.
- **Secrets**: never placed in `group_vars/all.yml` in plaintext. They go in a vault-encrypted
  `group_vars/vault.yml` (see `examples/inventory/group_vars/vault.yml.example`); `.gitignore`
  excludes `**/group_vars/vault.yml`.
- **SSH port changes**: `security_ssh_port` must be applied via `firewall.yml` *before*
  `security.yml` (open the new port first, then switch sshd to it), or the connection can be
  locked out. `site.yml`'s playbook order already encodes this.
- **Variable-driven roles, not parameters**: all three custom roles take a single list variable
  (`users`, `docker_apps`, `packages_base`/`packages_extra`) with `state: present|absent` (users)
  or `state: started|stopped|absent` (docker_apps) items; see each role's `defaults/main.yml` for
  the full item schema and comments — that's the source of truth for what fields each role reads.
- **docker_apps deploys files, then acts**: `docker_apps` copies `compose_src` (and optionally
  `env_file`) from the control machine into `docker_apps_base_dir` (default `/opt/docker-apps`)
  on the target host, then runs `community.docker.docker_compose_v2` (or `docker compose stop`
  for `state: stopped`, since that module doesn't support stop-without-down) from that deployed
  copy — it does not operate on `compose_src` directly on the remote.
