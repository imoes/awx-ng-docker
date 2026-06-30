# awx-ng

AWX fork with Foreman-style variable management for Ansible automation.

Based on [AWX 24.6.1](https://github.com/ansible/awx) (Apache 2.0).

## What's different from upstream AWX?

| Feature | AWX upstream | awx-ng |
|---|---|---|
| Role variable extraction | ✗ | ✓ Auto-scanned after every git sync |
| Per-host aggregated variables | ✗ | ✓ Merged view: role defaults → group vars → host vars |
| Survey generation from roles | ✗ | ✓ One click → survey spec from role defaults |
| Playbook editor | ✗ | ✓ Monaco-based file editor with YAML linting |
| Git integration in editor | ✗ | ✓ Commit & push directly from the UI |
| Site/Location management | ✗ | ✓ Sites with SSH credentials, env vars, ansible.cfg |
| Runner management | ✗ | ✓ Register remote execution nodes, assign to sites |
| MCP server | ✗ | ✓ JSON-RPC 2.0 endpoint for AI assistants |
| Root password hashing | ✗ | ✓ sha512crypt, stored in host vars |
| NetBox integration | ✗ | ✓ Location reconcile, inventory source ready |
| API token management | ✗ | ✓ Personal OAuth2 tokens (UI + API) |
| Ansible Vault Store | ✗ | ✓ Named vaults, auto-generated passwords, auto-injected at job start |
| SSO / Keycloak | optional | ✓ OIDC pre-configured via environment variables |

Full step-by-step documentation: **[WORKFLOW.md](WORKFLOW.md)**  
Remote runner setup: **[PROXIES.md](PROXIES.md)**

## Quickstart

```bash
git clone https://github.com/imoes/awx-ng-docker.git
cd awx-ng-docker

# 1. Generate secrets
mkdir -p secrets
python3 -c "import secrets; print(secrets.token_hex(32))" > secrets/secret_key
python3 -c "import secrets; print(secrets.token_hex(16))" > secrets/pg_password
chmod 600 secrets/*

# 2. Create configuration
cp .env.example .env
$EDITOR .env   # at minimum set AWX_ADMIN_PASSWORD and ANSIBLE_REPO_PATH

# 3. Create data directories
mkdir -p data/postgres data/redis data/projects
mkdir -p data/receptor && chmod 777 data/receptor

# 4. Build and start
docker compose build
docker compose up -d

# 5. Check status (web UI starts after ~60s)
docker compose ps
```

Web UI: **http://localhost:8052** — Login: `admin` / password from `.env`

## Requirements

- Docker ≥ 24
- Docker Compose v2
- 4 GB RAM, 20 GB free disk space
- An Ansible repository on the host (path in `ANSIBLE_REPO_PATH`)

## Architecture

```
Browser / API client / MCP client
        │
        ▼
  ┌─────────────┐
  │  awx_web    │  nginx + uWSGI · Port 8052 · REST API + Web UI + MCP endpoint
  └──────┬──────┘
         │ Job dispatch
         ▼
  ┌─────────────┐
  │  awx_task   │  Dispatcher · role-scan hook · credential injection
  └──────┬──────┘
         │ Receptor socket / mesh
         ▼
  ┌─────────────┐   ┌─────────────────────┐
  │   awx_ee    │   │  Remote runner       │  optional, dials out on port 2222
  │  (local)    │   │  (any Linux host)    │  (NAT-friendly)
  └─────────────┘   └─────────────────────┘
```

`AWX_DISABLE_CONTAINER_ISOLATION=True` — jobs run directly inside the container,
no nested Docker required.

## Configuration

### .env

| Variable | Required | Description |
|---|---|---|
| `AWX_ADMIN_PASSWORD` | ✓ | Admin password for the web UI |
| `ANSIBLE_REPO_PATH` | ✓ | Host path to the Ansible repo (mounted as `/var/lib/awx/projects/ansible03`) |
| `NETBOX_URL` | — | NetBox URL for location reconciliation |
| `NETBOX_TOKEN` | — | NetBox API token |
| `OIDC_ENDPOINT` | — | Keycloak endpoint (`https://.../auth/realms/<realm>`) |
| `OIDC_KEY` | — | Keycloak client ID |
| `OIDC_SECRET` | — | Keycloak client secret |
| `OIDC_VERIFY_SSL` | — | SSL verification (default: `true`) |

### config/custom.py

Django settings overlay — bind-mounted, no rebuild required.
Contains `INSTALLED_APPS`, `NETBOX_URL`, `AWX_DISABLE_CONTAINER_ISOLATION`, etc.

### Ansible repository mount

The Ansible repository (`ANSIBLE_REPO_PATH`) is bind-mounted into all three containers:

```yaml
- ${ANSIBLE_REPO_PATH}:/var/lib/awx/projects/ansible03:rw
```

Create an AWX project with SCM type **Manual** and playbook directory `ansible03`.

## Custom UI Screens

All awx-ng screens are integrated into the existing AWX navigation.

### Resources

| Screen | Path | Description |
|--------|------|-------------|
| Roles | `/roles` | Role overview with variables, tags, handlers; manual scan trigger |
| Playbooks | `/playbooks` | Playbook list with play details (hosts, roles, tags); launch jobs |
| Editor | `/editor` | File editor for playbooks and roles (Monaco + YAML linting + git) |
| Vaults | `/vaults` | Manage named ansible-vault stores (key-value pairs → encrypted YAML file); passwords auto-injected at job start |

### Administration

| Screen | Path | Description |
|--------|------|-------------|
| Runners | `/runner_sites` | Register execution nodes, assign to sites, health checks |
| Sites | `/locations` | Manage sites (= AWX instance groups); SSH credential + env + ansible.cfg per site |
| API Tokens | `/tokens` | Personal OAuth2 tokens for REST API and MCP |

### Editor features

- **File tree** with lazy-loading and context menu (rename, duplicate, delete, new file/folder)
- **Monaco YAML editor** with Jinja2 syntax highlighting
- **Real-time YAML linting** (800ms debounce) — errors highlighted inline
- **File upload** (ZIP/tar.gz auto-extracted)
- **Add mountpoint project** (`+` button): register an existing bind-mounted directory as an AWX project
- **Git panel** (git repos only): branch, dirty/ahead indicators, commit with message, push to origin

### Sites & Runners

Sites are linked 1:1 to AWX instance groups — creating a site automatically creates the instance group (and deleting a site removes it). System groups (`controlplane`, `default`) are protected.

**Runtime resolution:** AWX picks a runner from the site → per-runner override wins, otherwise the site default is used for SSH credential, environment variables, and ansible.cfg.

## Custom API Endpoints

Base URL: `http://localhost:8052/api/v2/`

### Role variables

```
GET  /api/v2/projects/{id}/role_variables/               # Variables from defaults/ + vars/
GET  /api/v2/projects/{id}/role_variables/?role_name=img_docker
POST /api/v2/projects/{id}/role_variables/scan/trigger/  # Trigger manual scan
GET  /api/v2/projects/{id}/role_tags/                    # Tags from tasks/**/*.yml
GET  /api/v2/projects/{id}/role_handlers/                # Handlers from handlers/main.yml
GET  /api/v2/projects/{id}/roles/                        # All roles (disk + DB)
```

### File editor

```
GET    /api/v2/projects/{id}/files/                      # Directory listing
GET    /api/v2/projects/{id}/files/content/?path=...     # Read file content
PUT    /api/v2/projects/{id}/files/content/?path=...     # Write file (+ auto git commit)
DELETE /api/v2/projects/{id}/files/content/?path=...     # Delete file
POST   /api/v2/projects/{id}/files/rename/               # Rename/move file (git mv)
POST   /api/v2/projects/{id}/files/upload/               # Upload file or archive
POST   /api/v2/projects/{id}/files/lint/                 # YAML validation
```

### Git operations

```
GET  /api/v2/projects/{id}/git/                          # Status, branch, log, ahead count
POST /api/v2/projects/{id}/git/  {"action": "commit", "message": "..."}
POST /api/v2/projects/{id}/git/  {"action": "push"}
```

### Playbook metadata

```
GET  /api/v2/projects/{id}/plays/                        # Playbook list with play details
GET  /api/v2/projects/{id}/plays/?playbook=site.yml      # Single playbook metadata
POST /api/v2/projects/{id}/launch/                       # Launch job (jt_id + limit + location_id)
GET  /api/v2/projects/{id}/variable_usages/?role=...&var=...  # Variable usage locations
```

### Host variables

```
GET    /api/v2/hosts/{id}/aggregated_variables/          # Merged variable stack
GET    /api/v2/hosts/{id}/role_variables/                # Host variables
PATCH  /api/v2/hosts/{id}/role_variables/{var_name}/     # Override value
DELETE /api/v2/hosts/{id}/role_variables/{var_name}/     # Reset to default
POST   /api/v2/hosts/{id}/assign_roles/                  # Assign roles to host
POST   /api/v2/hosts/{id}/run/                           # Launch job for this host
POST   /api/v2/hosts/{id}/set_root_password/             # Hash and store root password
```

### Sites & runners

```
GET/POST   /api/v2/locations/                            # List / create sites
PATCH/DEL  /api/v2/locations/{id}/                       # Edit / delete site
POST       /api/v2/locations/reconcile/                  # Sync with NetBox
GET/POST   /api/v2/execution_node_locations/             # Runner-site assignments
PATCH/DEL  /api/v2/execution_node_locations/{id}/
POST       /api/v2/runners/register/                     # Register runner
POST       /api/v2/runners/deprovision/                  # Remove runner
```

### Survey & tools

```
POST /api/v2/job_templates/{id}/generate_survey/         # Auto-generate survey from role defaults
POST /api/v2/tools/hash_password/                        # sha512crypt hash
```

### Ansible Vault Store

```
GET    /api/v2/vaults/                                   # List all vaults (no passwords/variables)
POST   /api/v2/vaults/                                   # Create vault (auto-generates password + AWX credential)
GET    /api/v2/vaults/{id}/                              # Get vault with plaintext variables
PATCH  /api/v2/vaults/{id}/                              # Update variables or description
DELETE /api/v2/vaults/{id}/                              # Delete vault + AWX credential
POST   /api/v2/vaults/{id}/generate/                     # Generate encrypted vault file
                                                         # Body: {"project_id": N} — writes to project dir
```

The vault password is **never** returned by the API. It is stored in the DB and injected
automatically as an AWX vault credential at job start. Reference the generated file in playbooks:

```yaml
vars_files:
  - vault-meine-vault.yml   # file is written by POST /api/v2/vaults/{id}/generate/
```

### Tokens & MCP

```
GET/POST   /api/v2/tokens/                               # Personal OAuth2 tokens
DELETE     /api/v2/tokens/{id}/
GET/POST   /mcp                                          # MCP server (JSON-RPC 2.0, Bearer auth)
```

## Rebuild after code changes

Custom code (`custom/`) is baked into the Docker image — rebuild on changes:

```bash
docker compose build awx_web awx_task
docker compose up -d awx_web awx_task
```

Config files (`config/`) are bind-mounted — `docker compose restart awx_web` is enough.

After migration changes:
```bash
docker compose run --rm awx_init awx-manage migrate customvars
```

## SSO / Keycloak

Keycloak setup (by the Keycloak admin):
1. Create client `awx-ng` (confidential)
2. Redirect URI: `http://<host>:8052/sso/complete/oidc/`
3. Enter client ID and secret in `.env` (`OIDC_KEY`, `OIDC_SECRET`, `OIDC_ENDPOINT`)

## MCP Server

The MCP server (Model Context Protocol) allows AI assistants to control AWX directly.

- Endpoint: `https://<host>/mcp` (JSON-RPC 2.0)
- Auth: Bearer token (OAuth2) or Django session
- Create tokens: **Resources → API Tokens** or `POST /api/v2/tokens/`
- Available tools: `awx_run_playbook`, `awx_list_inventories`, `awx_list_projects`,
  `awx_list_project_files`, `awx_read_project_file`, `awx_write_project_file`, and more

## Project structure

```
awx-ng-docker/
├── Dockerfile              # AWX 24.6.1 base + awx-ng patches
├── Dockerfile.ee           # Execution environment (ansible-runner + Receptor + collections)
├── Dockerfile.proxy        # Remote runner image (same toolchain as Dockerfile.ee)
├── docker-compose.yml      # Core stack: awx_web + awx_task + awx_ee
├── docker-compose.proxy.yml # Remote runner compose (for external hosts)
├── .env.example            # Configuration template
├── config/
│   ├── custom.py           # Django settings overlay (bind-mount, no rebuild)
│   └── nginx_awx.conf      # nginx configuration (bind-mount, no rebuild)
├── scripts/
│   ├── init_awx.sh                  # DB migration + admin user on first start
│   ├── setup-proxy.sh               # receptor.conf generator for remote runners
│   └── launch_awx_task_ng.sh        # Task container entrypoint
├── receptor/
│   ├── receptor-control.conf        # Receptor control node config (port 2222)
│   └── receptor-proxy.conf.template # Remote runner template
├── secrets/                # secret_key + pg_password (not in git)
├── data/                   # Runtime data (not in git)
└── custom/awx/
    ├── customvars/         # Django app: models, API, migrations, MCP tools
    ├── api/urls/           # Patched URL routes
    └── main/tasks/         # Patched jobs.py (credential injection, env vars)
```

## License

Apache License 2.0 — full text in [LICENSE](LICENSE), attributions in [NOTICE](NOTICE).

awx-ng is a fork of [ansible/awx](https://github.com/ansible/awx)
(Copyright Ansible, a Red Hat Company), licensed under Apache 2.0.
All modifications are also licensed under Apache 2.0.
