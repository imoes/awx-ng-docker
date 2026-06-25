# Remote Ansible-Runner (Execution Nodes)

awx-ng nutzt das **Receptor-Protokoll** für verteilte Job-Ausführung. Remote Runner
verbinden sich **ausgehend** (NAT-freundlich, kein eingehender Port nötig) zum zentralen
awx-ng Control Node auf Port 2222 und führen Playbooks lokal aus.

## Architektur

```
awx-ng (Port 8052)
  └─ awx_ee [lokal] (Port 2222)   ← Receptor Control Node
       ├─ ansible03.example.com   ← Remote Runner (Site: MUE-0)
       ├─ ansible01.example.com   ← Remote Runner (Site: Berlin)
       └─ ... weitere Runner
```

Jeder Runner wird in awx-ng einer **Site** zugeordnet. Job-Templates können dann
über die Site gesteuert werden, welcher Runner den Job ausführt.

**Wichtig:** AWX registriert Receptor Nodes **nicht** automatisch. Ein Runner, der sich
über Receptor verbindet, erscheint als `"Unrecognized node advertising on mesh"` in den
Logs und wird ignoriert, bis er in AWX registriert wurde (Schritt 4 unten).

## Docker (empfohlen)

Voraussetzungen: Docker + Docker Compose auf dem Remote-Host.

```bash
# 1. Dateien auf den Remote-Host kopieren (aus dem awx-ng-docker-Verzeichnis)
scp Dockerfile.proxy \
    docker-compose.proxy.yml \
    scripts/setup-proxy.sh \
    receptor/receptor-proxy.conf.template \
    user@ansible03:~/awx-runner/

# 2. Auf dem Remote-Host: receptor.conf generieren + Verzeichnisse anlegen
cd ~/awx-runner
./setup-proxy.sh ansible03 awx-ng.example.com 2222
#                ^^^^^^^^^  ^^^^^^^^^^^^^^^^^^  ^^^^
#                Node-ID    Control-Host         Port

# 3. Image bauen (installiert Ansible + Collections ins Image)
docker compose -f docker-compose.proxy.yml build

# 4. Container starten (Receptor verbindet sich ausgehend zu Port 2222)
docker compose -f docker-compose.proxy.yml up -d

# 5. In AWX UI registrieren: Administration → Runners → "Register runner"
#    Hostname: ansible03 (identisch mit dem Node-ID aus setup-proxy.sh)
#    Node type: execution
#    → AWX legt den Instance-Datensatz an

# 6. Health check klicken (oder ~20s warten)
#    → AWX erkennt den Receptor Node im Mesh → node_state=ready, Capacity > 0
```

Für Hosts hinter einem Corporate-HTTP-Proxy: `docker-compose.override.yml` anlegen
(gitignored) mit `HTTP_PROXY` / `HTTPS_PROXY` Build-Args.

## Runner einer Site zuweisen

Nach der Registrierung in AWX den Runner einer Site zuordnen:

1. **Administration → Runners** → Zeile des neuen Runners → **assign site**
2. Site auswählen (z.B. `MUE-0`)
3. Optional: SSH-Credential, Umgebungsvariablen, ansible.cfg **als Override** eintragen
   (leer = Site-Default wird verwendet)
4. Save

Die Site legt automatisch eine AWX Instance Group an und der Runner wird ihr zugewiesen.

## Manuelle Installation (ohne Docker)

Voraussetzungen auf dem Remote-Host:
- Linux (Debian/Ubuntu/RHEL)
- Python 3.8+
- Ausgehende TCP-Verbindung zu awx-ng-Control-Host:2222

### Via Ansible-Playbook

```bash
ansible-playbook deploy/ansible/install-receptor-proxy.yml \
  -i "ansible03.example.com," \
  -e "proxy_node_id=ansible03" \
  -e "awx_ng_control_host=awx-ng.example.com" \
  -e "awx_ng_control_port=2222" \
  --become
```

### Manuell

```bash
# 1. Receptor installieren
curl -Lo /usr/local/bin/receptor \
  https://github.com/ansible/receptor/releases/latest/download/receptor_linux_amd64
chmod +x /usr/local/bin/receptor

# 2. ansible-runner + Ansible installieren
pip3 install ansible-runner "ansible==8.7.0" netaddr proxmoxer

# 3. Receptor-Konfiguration anlegen
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

# 4. Systemd-Service anlegen
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

Danach in AWX UI registrieren: **Administration → Runners → Register runner**,
Hostname = Node-ID aus der receptor.conf (`ansible03`).

## Bekannte Runner

| Hostname | Node-ID | Site |
|----------|---------|------|
| ansible03.example.com | ansible03 | MUE-0 |

(Diese Tabelle nach Registrierung in awx-ng aktuell halten.)
