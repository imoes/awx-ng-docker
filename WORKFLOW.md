# awx-ng — Workflow Documentation

Complete walkthrough from initial setup to running a playbook.

---

## Part 1 — Initial setup (one time)

### 1.1 Prerequisites

- Docker ≥ 24, Docker Compose v2
- At least 4 GB RAM, 20 GB free disk space
- A local Ansible repository on the host (e.g. `/home/user/ansible/ansible03`)
- Outbound TCP connection on port 2222 (for remote runners, optional)

### 1.2 Create configuration

```bash
cd awx-ng-docker/

# Generate secrets (one time)
mkdir -p secrets
python3 -c "import secrets; print(secrets.token_hex(32))" > secrets/secret_key
python3 -c "import secrets; print(secrets.token_hex(16))" > secrets/pg_password
chmod 600 secrets/*

# Create configuration from template
cp .env.example .env
```

Fill in `.env`:
```bash
AWX_ADMIN_PASSWORD=MySecurePassword   # web UI login password

# Path to the Ansible repository on the HOST (not inside the container)
ANSIBLE_REPO_PATH=/home/user/ansible/ansible03

# Optional: NetBox integration for location reconciliation
NETBOX_URL=https://netbox.example.com
NETBOX_TOKEN=<api-token>
```

### 1.3 Create directories and start

```bash
mkdir -p data/postgres data/redis data/projects
mkdir -p data/receptor && chmod 777 data/receptor

docker compose build
docker compose up -d

# Check status — web UI starts after approximately 60–90 seconds
docker compose ps
```

**Web UI:** `http://localhost:8052`
**Login:** `admin` / password from `.env`

---

## Part 2 — AWX base configuration (one time)

### 2.1 Create an organization

AWX requires an organization as a container for inventories and projects.

1. **Resources → Organizations → Add**
2. Name: e.g. `IT` or your company name
3. Save

### 2.2 Create an inventory

The inventory contains the hosts and groups to run playbooks against.

1. **Resources → Inventories → Add → Add inventory**
2. Name: e.g. `ansible03-inventory`
3. Organization: the one created in 2.1
4. Save

### 2.3 Create a project

The Ansible repository (`ANSIBLE_REPO_PATH`) is mounted as `/var/lib/awx/projects/ansible03`
inside the container.

**Option A — Via the Editor (recommended):**

1. **Resources → Editor**
2. Click the `+` button next to the project dropdown
3. Enter a name (e.g. `ansible03`)
4. Select `ansible03` from the directory dropdown
5. Create

**Option B — Via the Projects screen:**

1. **Resources → Projects → Add**
2. Name: e.g. `ansible03`
3. Source Control Type: **Manual**
4. Playbook Directory: `ansible03`
   (directory name relative to `/var/lib/awx/projects/`)
5. Save

---

## Part 3 — Import hosts

### 3.1 Add a host manually

1. **Resources → Hosts → Add**
2. Name: hostname (e.g. `docker01.example.com`)
3. Inventory: the one created in 2.2
4. Variables (YAML):
   ```yaml
   ansible_host: 10.32.1.50
   ansible_user: root
   ```
5. Save

Alternatively: import hosts via a NetBox inventory source
(Resources → Inventories → Sources → Add, Source: `NetBox`).

### 3.2 Manage host variables

1. **Resources → Hosts → [select host]**
2. Tab **Role Variables**
3. All role defaults from the project are listed
4. Overridden values are shown in bold
5. Changes are saved directly to `host.variables` (equivalent to `host_vars`)

The **Aggregated Variables** tab shows the final variable stack:
role default → group override → host override (last one wins)

### 3.3 Assign roles to a host

Tab **Role Variables** → "Assign Roles" → select roles from the project.
Assigned roles are saved as `host_roles`.

---

## Part 4 — Explore playbooks and roles

### 4.1 Roles overview

**Resources → Roles**

Lists all roles in the project with variable counts, status (on disk / imported),
and last scan status. Click a role to see all variables with types and defaults.

Expand variable usages: where is the variable defined and used in the codebase?

### 4.2 Playbooks overview

**Resources → Playbooks**

All `.yml` files in the project with their plays (hosts, roles, tags).

- **Green badge + "Run" button**: a job template exists → launch directly
- **"Create template"**: no template yet → redirects to template creation

### 4.3 Edit files in the Editor

**Resources → Editor**

- File tree on the left, Monaco editor on the right
- Context menu (right-click): rename, duplicate, delete, new file/folder, upload
- Automatic YAML linting with inline error display
- **Git panel** (toolbar button, git repos only): commit and push the current state

---

## Part 5 — Create a job template

### 5.1 Add a template

1. **Resources → Templates → Add → Add job template**
2. Name: e.g. `Deploy Docker Host`
3. Job Type: `Run`
4. Inventory: the one created in 2.2
5. Project: the one created in 2.3
6. Playbook: select from dropdown
7. Credentials: select a machine credential with an SSH key (if required)
8. **Limit**: check "Prompt on launch" → limit is requested at each launch
9. **Site/Runner** (optional): select a site → job runs on that site's runner
10. Save

### 5.2 Optional settings

- **Tags** / **Skip Tags**: run only specific tasks
- **Extra Variables**: static overrides for all jobs from this template
- **Verbosity**: log detail level (0 = normal, 2 = debug)

---

## Part 6 — Run a playbook

### Method A: Playbooks screen (simplest)

1. **Resources → Playbooks**
2. Playbook row with green badge → click **Run**
3. In the dialog:
   - **Template**: select if multiple exist
   - **Limit**: hostname or group (empty = all hosts in inventory)
   - **Location**: select a site → job runs on that site's runner
4. Click **Run**

### Method B: Host screen

1. **Resources → Hosts → [select host]**
2. Tab **Role Variables** → click **Run Host**
3. Limit is pre-filled with the hostname

### Method C: Templates (standard AWX)

1. **Resources → Templates → [select template] → Launch**

### 6.1 What happens when a job runs

```
AWX (awx_task)
  │
  ├── Generates private data directory:
  │     project/    → playbooks + roles from the Ansible repository
  │     inventory/  → inventory file with host_vars from the AWX database
  │     env/        → credentials, limit, extra vars, proxy env vars
  │
  ├── Sends the directory as a tarball via Receptor to the execution node
  │
  └── Execution node (local awx_ee or remote runner):
        ansible-runner worker
          → ansible-playbook site.yml --limit docker01.example.com
```

### 6.2 Monitor job output

**Views → Jobs** → running or completed job → live output with color coding.

---

## Part 7 — Set up a remote runner (optional)

A remote runner executes jobs locally on another host (closer to the target hosts).
It dials **outbound** to the awx-ng control node on port 2222.

Full instructions: **[PROXIES.md](PROXIES.md)**

### Quick reference

```bash
# 1. Copy files to remote host + build image
scp Dockerfile.proxy docker-compose.proxy.yml scripts/setup-proxy.sh \
    receptor/receptor-proxy.conf.template user@runner:~/awx-runner/
ssh user@runner "cd ~/awx-runner && \
  ./setup-proxy.sh runner-hostname awx-ng.example.com 2222 && \
  docker compose -f docker-compose.proxy.yml build && \
  docker compose -f docker-compose.proxy.yml up -d"

# 2. Register in AWX: Administration → Runners → Register runner
#    Hostname = runner-hostname (must match node ID)

# 3. Assign to site: Administration → Runners → assign site
```

---

## Part 8 — NetBox integration (optional)

### 8.1 Import sites from NetBox

1. **Administration → Sites → Reconcile with NetBox**
2. `NETBOX_URL` and `NETBOX_TOKEN` must be set in `.env`
3. All NetBox sites are created as awx-ng sites

### 8.2 Import hosts from NetBox

1. **Resources → Inventories → [inventory] → Sources → Add**
2. Source: `NetBox`
3. Enter NetBox URL and credentials
4. Start sync

---

## Part 9 — MCP server (AI integration, optional)

The MCP server allows AI assistants (e.g. Claude) to control awx-ng directly.

### 9.1 Create a token

1. **Resources → API Tokens → Create Token**
2. Scope: `write`
3. Copy the token immediately — it is only shown once

### 9.2 Configure MCP in Claude Code

```json
{
  "mcpServers": {
    "awx-ng": {
      "command": "curl",
      "args": ["-s", "-X", "POST", "http://localhost:8052/mcp"],
      "env": {
        "AWX_TOKEN": "<your-token>"
      }
    }
  }
}
```

### 9.3 Available MCP tools

| Tool | Description |
|------|-------------|
| `awx_list_projects` | List projects |
| `awx_list_inventories` | List inventories |
| `awx_run_playbook` | Launch a playbook |
| `awx_list_project_files` | List files in a project |
| `awx_read_project_file` | Read file content |
| `awx_write_project_file` | Write file content |
| `awx_sync_project` | Trigger git sync |
| `awx_get_project_roles` | List roles |
| `awx_get_role_variables` | Read role variables |

---

## Summary: minimal workflow

```
1. Fill in .env (ANSIBLE_REPO_PATH + AWX_ADMIN_PASSWORD)
2. docker compose build && docker compose up -d
3. UI: create organization
4. UI: create inventory
5. UI: Editor → "+" → create mountpoint project (directory: ansible03)
6. UI: add host + set host_vars
7. UI: Playbooks → "Create template" for the desired playbook
8. UI: Playbooks → "Run" → enter limit → Run
9. UI: Views → Jobs → watch output
```
