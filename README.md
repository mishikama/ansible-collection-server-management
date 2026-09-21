# Ansible: managing Debian/Ubuntu servers

The project is structured around YAML inventory and variable files — all
management (users, firewall, packages, docker apps) is described
declaratively in `group_vars/`, without touching role code.

The inventory and docker-compose files currently in the repository
(`examples/inventory/`, `examples/files/`) are a **working example**, not
your real configuration. For actual use, copy `examples/inventory/` into
your own `inventory/` (don't commit sensitive data — hosts, keys — into the
shared repo) and fill in your own values; the playbooks and `ansible.cfg`
aren't hard-wired to this — the inventory path can be overridden with a
single flag (`-i`) or a line in `ansible.cfg`.

## What this project does

| Area | Role | Source |
|---|---|---|
| SSH hardening, sudoers, fail2ban, security auto-updates | `geerlingguy.security` | public role (Galaxy) |
| Firewall (iptables, list of allowed ports) | `geerlingguy.firewall` | public role (Galaxy) |
| Install Docker Engine + `docker compose` plugin | `geerlingguy.docker` | public role (Galaxy) |
| Install packages, `apt dist-upgrade`, auto-reboot | `mishikama.server_management.packages` | own, in the collection (no suitable public one) |
| Users: create/remove, ssh keys, sudo | `mishikama.server_management.users` | own, in the collection |
| Deploy and manage docker-compose stacks (start/stop/remove) | `mishikama.server_management.docker_apps` | own, in the collection |

The three custom roles are packaged as a separate **Ansible Collection**
(`ansible-collection-server-management/`, namespace `mishikama.server_management`) — so they can
be reused in other projects with a single line in `requirements.yml`,
no copy-pasting code. Details — `ansible-collection-server-management/README.md`.

## Installing dependencies

```bash
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml
```

The `mishikama.server_management` collection itself doesn't need a separate
install — it's already resolved locally via
`.collections/ansible_collections/mishikama/server_management`
(a symlink to `ansible-collection-server-management/`, configured in
`ansible.cfg` → `collections_path`). Edits in
`ansible-collection-server-management/roles/*` are immediately visible to
the playbooks.

## Inventory

Example — `examples/inventory/hosts.yml`. Copy `examples/inventory/` into
your own `inventory/` (or use `examples/` directly while testing) and set
your hosts in the `debian_servers` (all servers) and `docker_hosts` (the
ones that need Docker) groups:

```bash
cp -r examples/inventory inventory
```

Then update the path in `ansible.cfg` (`inventory = ...`) or pass it
explicitly: `ansible-playbook playbooks/site.yml -i inventory/hosts.yml`.

## Variables

Example — `examples/inventory/group_vars/all.yml` (after copying —
`inventory/group_vars/all.yml`). It configures:

- `security_*` — SSH port, root login, password authentication, etc.
- `firewall_extra_tcp_ports` / `firewall_allowed_udp_ports` — allowed ports besides
  SSH (the SSH port from `security_ssh_port` is added automatically — see
  `playbooks/firewall.yml`, no need to list it yourself)
- `packages_base` / `packages_extra` — which packages to install
- `users` — list of users (create/remove, sudo, ssh keys)
- `docker_apps` — list of docker-compose applications and their desired state

Don't store secrets (password hashes, etc.) in plain text — see
`examples/inventory/group_vars/vault.yml.example` and `ansible-vault`.

For individual hosts/groups you can override variables in
`inventory/host_vars/<host>.yml` or create new files under `group_vars/`.

## Running it

```bash
# everything at once
ansible-playbook playbooks/site.yml

# piece by piece
ansible-playbook playbooks/packages.yml   # packages + system update
ansible-playbook playbooks/security.yml   # ssh hardening, auto-updates, fail2ban
ansible-playbook playbooks/firewall.yml   # firewall
ansible-playbook playbooks/users.yml      # users and ssh keys
ansible-playbook playbooks/docker.yml     # install docker
ansible-playbook playbooks/apps.yml       # deploy/manage docker-compose applications

# dry run
ansible-playbook playbooks/site.yml --check --diff

# limit to one host
ansible-playbook playbooks/site.yml --limit srv1.example.com
```

## Changing the SSH port without locking yourself out

1. Change `security_ssh_port` in `all.yml`.
2. Run `firewall.yml` **first** (opens the new port), then `security.yml`
   (applies the new port in sshd) — in exactly this order, otherwise you
   risk losing access.
3. Update `ansible_port` in the inventory / your own `~/.ssh/config` for
   subsequent connections.

## Adding / removing a user

Edit the `users` list in your `inventory/group_vars/all.yml` (or `host_vars`):

```yaml
users:
  - name: ivan
    state: present
    sudo: true
    sudo_nopasswd: false
    ssh_keys:
      - "ssh-ed25519 AAAA... ivan@laptop"

  - name: staff_leaving
    state: absent   # removes the user and their home directory
```

Then:

```bash
ansible-playbook playbooks/users.yml
```

## Adding your own docker-compose application

`examples/files/docker-compose/` has three ready-made examples for
different cases:

| Example | Demonstrates |
|---|---|
| `example-app/` | minimal stack (one service, one port) |
| `app-with-env/` | environment variables via `.env` (see the `.env.example` next to it) |
| `app-with-volume/` | named volume + config bind-mount (nginx.conf) |

To add your own application:

1. Put `docker-compose.yml` (and `.env`, configs if needed) into
   `files/docker-compose/<app-name>/` (in your own copy, not in `examples/`).
2. Add the application to `docker_apps` in `inventory/group_vars/all.yml`
   (in `examples/inventory/group_vars/all.yml` three examples are already
   there, two of them commented out):

```yaml
docker_apps:
  - name: my-service
    state: started        # started | stopped | absent
    pull: missing          # always | missing | never
    remove_orphans: true
    compose_src: "{{ playbook_dir }}/../files/docker-compose/my-service"
    # env_file: "{{ playbook_dir }}/../files/docker-compose/my-service/.env"
    # remove_volumes: false   # only for state: absent
```

3. Apply it:

```bash
ansible-playbook playbooks/apps.yml
```

To stop an application without deleting its data/volume — set
`state: stopped`. To tear the stack down completely (`docker compose down`)
— `state: absent` (add `remove_volumes: true` if you also want the data gone).

## Structure

```
ansible/
├── ansible.cfg
├── requirements.yml                      # public geerlingguy roles + collections
├── examples/                             # working example: copy it and fill in your own data
│   ├── inventory/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       ├── all.yml                   # variables with examples for every role
│   │       └── vault.yml.example         # secrets template
│   └── files/docker-compose/
│       ├── example-app/                  # minimal stack
│       ├── app-with-env/                 # stack with .env
│       └── app-with-volume/              # stack with a volume + config bind-mount
├── playbooks/
│   ├── site.yml                          # full run
│   ├── packages.yml
│   ├── security.yml
│   ├── firewall.yml
│   ├── users.yml
│   ├── docker.yml
│   └── apps.yml
├── roles/                                # public roles (geerlingguy.*), installed via galaxy
├── ansible-collection-server-management/ # own reusable collection mishikama.server_management
│   ├── galaxy.yml
│   ├── README.md
│   ├── meta/runtime.yml
│   └── roles/
│       ├── users/
│       ├── packages/
│       └── docker_apps/
└── .collections/                         # symlink for locally resolving mishikama.server_management
```

## How to extract the collection into a separate public repository

`ansible-collection-server-management/` is self-contained — `galaxy.yml`
already sits at its root. When you want to reuse the roles in another
project:

```bash
cd ansible-collection-server-management
git init && git add . && git commit -m "Initial commit"
git remote add origin https://github.com/<account>/ansible-collection-server-management.git
git push -u origin main
```

In the other project, just add this to its `requirements.yml`:

```yaml
collections:
  - name: https://github.com/<account>/ansible-collection-server-management.git
    type: git
    version: main
```
