# Remote Ansible Runners (Execution Nodes)

awx-ng uses the **Receptor protocol** for distributed job execution. Remote runners
dial **outbound** (NAT-friendly, no inbound port required) to the central awx-ng
control node on port 2222 and execute playbooks locally.

## Architecture

```
awx-ng (port 8052)
  └─ awx_ee [local] (port 2222)   ← Receptor control node
       ├─ ansible03.example.com   ← Remote runner (site: MUE-0)
       ├─ ansible01.example.com   ← Remote runner (site: Berlin)
       └─ ... more runners
```

Each runner is assigned to a **site** in awx-ng. Job templates can then be directed
to a specific site to control which runner executes the job.

**Important:** AWX does **not** register Receptor nodes automatically. A runner that
connects via Receptor appears as `"Unrecognized node advertising on mesh"` in the logs
and is ignored until it is registered in AWX (step 5 below).

## Docker (recommended)

Prerequisites: Docker + Docker Compose on the remote host.

```bash
# 1. Copy files to the remote host (from the awx-ng-docker directory)
scp Dockerfile.proxy \
    docker-compose.proxy.yml \
    scripts/setup-proxy.sh \
    receptor/receptor-proxy.conf.template \
    user@ansible03:~/awx-runner/

# 2. On the remote host: generate receptor.conf and create directories
cd ~/awx-runner
./setup-proxy.sh ansible03 awx-ng.example.com 2222
#                ^^^^^^^^^  ^^^^^^^^^^^^^^^^^^  ^^^^
#                node ID    control host         port

# 3. Build the image (installs Ansible + collections)
docker compose -f docker-compose.proxy.yml build

# 4. Start the container (Receptor dials out to port 2222)
docker compose -f docker-compose.proxy.yml up -d

# 5. Register in the AWX UI: Administration → Runners → "Register runner"
#    Hostname: ansible03 (must match the node ID from setup-proxy.sh)
#    Node type: execution
#    → AWX creates the instance record

# 6. Click "health check" (or wait ~20s)
#    → AWX detects the Receptor node in the mesh → node_state=ready, capacity > 0
```

For hosts behind a corporate HTTP proxy: create a `docker-compose.override.yml`
(gitignored) with `HTTP_PROXY` / `HTTPS_PROXY` build args.

## Assign runner to a site

After registration, assign the runner to a site:

1. **Administration → Runners** → row of the new runner → **assign site**
2. Select a site (e.g. `MUE-0`)
3. Optionally enter SSH credential, environment variables, ansible.cfg as **overrides**
   (leave empty to use the site default)
4. Save

The site automatically creates an AWX instance group and the runner is assigned to it.

## Manual installation (without Docker)

Prerequisites on the remote host:
- Linux (Debian/Ubuntu/RHEL)
- Python 3.8+
- Outbound TCP connection to awx-ng-control-host:2222

### Via Ansible playbook

```bash
ansible-playbook deploy/ansible/install-receptor-proxy.yml \
  -i "ansible03.example.com," \
  -e "proxy_node_id=ansible03" \
  -e "awx_ng_control_host=awx-ng.example.com" \
  -e "awx_ng_control_port=2222" \
  --become
```

### Manual steps

```bash
# 1. Install Receptor
curl -Lo /usr/local/bin/receptor \
  https://github.com/ansible/receptor/releases/latest/download/receptor_linux_amd64
chmod +x /usr/local/bin/receptor

# 2. Install ansible-runner and Ansible
pip3 install ansible-runner "ansible==8.7.0" netaddr proxmoxer

# 3. Create Receptor configuration
mkdir -p /etc/receptor /var/run/receptor
cat > /etc/receptor/receptor.conf << EOF
---
- node:
    id: ansible03

- log-level: info

- tcp-peer:
    address: awx-ng.example.com:2222
    redial: true

- control-service:
    service: control
    filename: /var/run/receptor/receptor.sock

- work-command:
    worktype: ansible-runner
    command: ansible-runner
    params: worker
    allowruntimeparams: true
    verifysignature: false
EOF

# 4. Create systemd service
cat > /etc/systemd/system/receptor.service << 'EOF'
[Unit]
Description=Receptor — AWX-NG Execution Node
After=network.target

[Service]
ExecStart=/usr/local/bin/receptor --config /etc/receptor/receptor.conf
Restart=always
RestartSec=10
User=awx

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now receptor
```

Then register in the AWX UI: **Administration → Runners → Register runner**,
hostname = node ID from receptor.conf (`ansible03`).

## Known runners

| Hostname | Node ID | Site |
|----------|---------|------|
| ansible03.example.com | ansible03 | MUE-0 |

(Keep this table up to date after registering runners in awx-ng.)
