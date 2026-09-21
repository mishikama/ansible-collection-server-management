# mishikama.server_management

Ansible Collection with roles for basic Debian/Ubuntu server management.

## Roles

| Role | FQCN | Purpose |
|---|---|---|
| `users` | `mishikama.server_management.users` | Create/remove users, ssh keys, sudo |
| `packages` | `mishikama.server_management.packages` | Install packages, `apt dist-upgrade`, auto-update |
| `docker_apps` | `mishikama.server_management.docker_apps` | Deploy and manage docker-compose stacks (started/stopped/absent) |

Variables for each role and the expected input format — see `roles/<role>/defaults/main.yml`.

## Installing in another project

### Option A — from git (while the repository isn't on Galaxy)

In the `requirements.yml` of the project that wants to reuse these roles:

```yaml
collections:
  - name: https://github.com/<your-account>/ansible-collection-server-management.git
    type: git
    version: main   # or a tag/commit for stability
```

```bash
ansible-galaxy collection install -r requirements.yml
```

### Option B — from Ansible Galaxy (after publishing)

```yaml
collections:
  - name: mishikama.server_management
    version: ">=1.0.0"
```

## Using it in a playbook

```yaml
- hosts: debian_servers
  become: true
  roles:
    - role: mishikama.server_management.users
    - role: mishikama.server_management.packages
    - role: mishikama.server_management.docker_apps
```

## How to publish this as a separate public repository

This folder (`ansible-collection-server-management/`) is self-contained:
`galaxy.yml` already sits at its root, as required for installing via
`type: git`. To extract it into its own repository as-is:

```bash
cd ansible-collection-server-management
git init
git add .
git commit -m "Initial commit: mishikama.server_management collection"
git remote add origin https://github.com/<your-account>/ansible-collection-server-management.git
git push -u origin main
```

After that you can (optionally) import it on
[galaxy.ansible.com](https://galaxy.ansible.com) — then installing it works
via option B (just `mishikama.server_management`, no git URL needed).

## Local development (within this same repository)

The main project (`../`) is already set up to use this collection locally —
see `../ansible.cfg` (`collections_path`) and the symlink
`../.collections/ansible_collections/mishikama/server_management`. Edit the
roles here — changes are immediately visible to the playbooks in
`../playbooks/*.yml`, no git repository needed.
