# awx-ng — Workflow-Dokumentation

Vollständiger Ablauf von der Ersteinrichtung bis zum Ausführen eines Playbooks.

---

## Teil 1 — Ersteinrichtung (einmalig)

### 1.1 Voraussetzungen

- Docker ≥ 24, Docker Compose v2
- Min. 4 GB RAM, 20 GB freier Speicher
- Ein lokales Ansible-Repository auf dem Host (z.B. `/home/user/ansible/ansible03`)
- Ausgehende TCP-Verbindung auf Port 2222 (für Remote Runner, optional)

### 1.2 Konfiguration anlegen

```bash
cd awx-ng-docker/

# Secrets generieren (einmalig)
mkdir -p secrets
python3 -c "import secrets; print(secrets.token_hex(32))" > secrets/secret_key
python3 -c "import secrets; print(secrets.token_hex(16))" > secrets/pg_password
chmod 600 secrets/*

# Konfiguration aus Vorlage ableiten
cp .env.example .env
```

`.env` befüllen:
```bash
AWX_ADMIN_PASSWORD=MeinSicheresPasswort   # Login für die Web-UI

# Pfad zum Ansible-Repository auf dem HOST (nicht im Container)
ANSIBLE_REPO_PATH=/home/user/ansible/ansible03

# Optional: NetBox-Anbindung für Location-Reconcile
NETBOX_URL=https://netbox.example.com
NETBOX_TOKEN=<api-token>
```

### 1.3 Verzeichnisse anlegen und starten

```bash
mkdir -p data/postgres data/redis data/projects
mkdir -p data/receptor && chmod 777 data/receptor

docker compose build
docker compose up -d

# Status prüfen — Web-UI startet nach ca. 60–90 Sekunden
docker compose ps
```

**Web-UI:** `http://localhost:8052`
**Login:** `admin` / Passwort aus `.env`

---

## Teil 2 — AWX-Grundkonfiguration (einmalig)

### 2.1 Organisation anlegen

AWX benötigt zwingend eine Organisation als Container für Inventories und Projekte.

1. **Resources → Organizations → Add**
2. Name: z.B. `IT` oder Firmenname
3. Save

### 2.2 Inventory anlegen

Das Inventory enthält die Hosts und Gruppen.

1. **Resources → Inventories → Add → Add inventory**
2. Name: z.B. `ansible03-inventory`
3. Organization: die in 2.1 erstellte
4. Save

### 2.3 Projekt anlegen

Das Ansible-Repository (`ANSIBLE_REPO_PATH`) ist als `/var/lib/awx/projects/ansible03`
im Container eingehängt.

**Option A — Über den Editor (empfohlen):**

1. **Resources → Editor**
2. `+`-Button neben dem Projekt-Dropdown klicken
3. Name eingeben (z.B. `ansible03`)
4. Verzeichnis `ansible03` aus dem Dropdown wählen
5. Create

**Option B — Über den Projects-Screen:**

1. **Resources → Projects → Add**
2. Name: z.B. `ansible03`
3. Source Control Type: **Manual**
4. Playbook Directory: `ansible03`
   (Verzeichnisname relativ zu `/var/lib/awx/projects/`)
5. Save

---

## Teil 3 — Hosts importieren

### 3.1 Host manuell anlegen

1. **Resources → Hosts → Add**
2. Name: Hostname (z.B. `docker01.example.com`)
3. Inventory: das in 2.2 erstellte
4. Variables (YAML):
   ```yaml
   ansible_host: 10.32.1.50
   ansible_user: root
   ```
5. Save

Alternativ: Hosts über NetBox-Inventory-Source importieren
(Resources → Inventories → Sources → Add, Source: `NetBox`).

### 3.2 Host-Variablen pflegen

1. **Resources → Hosts → [Host auswählen]**
2. Tab **Role Variables**
3. Alle Rollen-Defaults aus dem Projekt sind aufgelistet
4. Überschriebene Werte fett hervorgehoben
5. Änderungen direkt in `host.variables` gespeichert (= `host_vars`)

Tab **Aggregated Variables** zeigt den finalen Variablen-Stack:
Rollen-Default → Gruppen-Override → Host-Override (letzteres gewinnt)

### 3.3 Rollen dem Host zuweisen

Tab **Role Variables** → „Assign Roles" → Rollen aus dem Projekt auswählen.
Zugewiesene Rollen werden als `host_roles` gespeichert.

---

## Teil 4 — Playbooks und Rollen erkunden

### 4.1 Rollen-Übersicht

**Resources → Roles**

Zeigt alle Rollen des Projekts mit Variablen-Anzahl, Status (auf Disk / importiert)
und letztem Scan-Status. Klick auf eine Rolle → Variablen-Übersicht mit Types und Defaults.

Variablen-Verwendungen expandieren: wo ist die Variable im Code definiert und genutzt?

### 4.2 Playbook-Übersicht

**Resources → Playbooks**

Alle `.yml`-Dateien des Projekts mit den enthaltenen Plays (Hosts, Rollen, Tags).

- **Grüner Badge + „Run"-Button**: Job-Template vorhanden → direkt starten
- **„Create template"**: noch kein Template → leitet zur Template-Erstellung weiter

### 4.3 Dateien im Editor bearbeiten

**Resources → Editor**

- Datei-Baum links, Monaco-Editor rechts
- Kontextmenü (Rechtsklick): umbenennen, duplizieren, löschen, neue Datei/Ordner, Upload
- Automatisches YAML-Linting mit Inline-Fehleranzeige
- **Git-Panel** (Button in der Toolbar, nur für Git-Repos): aktuellen Stand committen und pushen

---

## Teil 5 — Job Template erstellen

### 5.1 Template anlegen

1. **Resources → Templates → Add → Add job template**
2. Name: z.B. `Deploy Docker Host`
3. Job Type: `Run`
4. Inventory: das in 2.2 erstellte
5. Project: das in 2.3 erstellte
6. Playbook: aus Dropdown wählen
7. Credentials: Machine Credential mit SSH-Key (falls benötigt)
8. **Limit**: Haken bei „Prompt on launch" → bei jedem Start wird Limit abgefragt
9. **Site/Runner** (optional): Site aus Dropdown wählen → Job läuft auf diesem Runner
10. Save

### 5.2 Optionale Einstellungen

- **Tags** / **Skip Tags**: nur bestimmte Tasks ausführen
- **Extra Variables**: statische Overrides für alle Jobs dieses Templates
- **Verbosity**: Log-Detail-Level (0 = normal, 2 = debug)

---

## Teil 6 — Playbook ausführen

### Weg A: Über den Playbooks-Screen (einfachster Weg)

1. **Resources → Playbooks**
2. Playbook-Zeile mit grünem Badge → **Run** klicken
3. Im Dialog:
   - **Template**: auswählen falls mehrere existieren
   - **Limit**: Hostname oder Gruppe (leer = alle Hosts im Inventory)
   - **Location**: Site auswählen → Job läuft auf dem Runner dieser Site
4. **Run** klicken

### Weg B: Über den Host-Screen

1. **Resources → Hosts → [Host auswählen]**
2. Tab **Role Variables** → **Run Host**-Button
3. Limit ist bereits mit dem Hostnamen vorbelegt

### Weg C: Über Templates (AWX-Standard)

1. **Resources → Templates → [Template auswählen] → Launch**

### 6.1 Was passiert beim Ausführen

```
AWX (awx_task)
  │
  ├── Generiert Private Data Directory:
  │     project/    → Playbooks + Rollen aus dem Ansible-Repository
  │     inventory/  → Inventory-Datei mit host_vars aus der AWX-Datenbank
  │     env/        → Credentials, Limit, Extra-Vars, Proxy-Env-Vars
  │
  ├── Sendet das Verzeichnis als Tarball via Receptor an den Execution Node
  │
  └── Execution Node (awx_ee lokal oder Remote Runner):
        ansible-runner worker
          → ansible-playbook site.yml --limit docker01.example.com
```

### 6.2 Job-Output beobachten

**Views → Jobs** → laufender oder abgeschlossener Job → Live-Output mit Farbkodierung.

---

## Teil 7 — Remote Runner einrichten (optional)

Ein Remote Runner führt Jobs lokal auf einem anderen Host aus (näher an den Ziel-Hosts).
Er verbindet sich **ausgehend** zum awx-ng Control Node auf Port 2222.

Vollständige Anleitung: **[PROXIES-de.md](PROXIES-de.md)**

### Kurzfassung

```bash
# 1. Dateien auf Remote-Host kopieren + Image bauen
scp Dockerfile.proxy docker-compose.proxy.yml scripts/setup-proxy.sh \
    receptor/receptor-proxy.conf.template user@runner:~/awx-runner/
ssh user@runner "cd ~/awx-runner && ./setup-proxy.sh runner-hostname awx-ng.example.com 2222 && docker compose -f docker-compose.proxy.yml build && docker compose -f docker-compose.proxy.yml up -d"

# 2. In AWX registrieren: Administration → Runners → Register runner
#    Hostname = runner-hostname (identisch mit Node-ID)

# 3. Site zuweisen: Administration → Runners → assign site
```

---

## Teil 8 — NetBox-Integration (optional)

### 8.1 Sites aus NetBox importieren

1. **Administration → Sites → Reconcile with NetBox**
2. `NETBOX_URL` und `NETBOX_TOKEN` müssen in `.env` gesetzt sein
3. Alle NetBox-Sites werden als awx-ng Sites angelegt

### 8.2 Hosts aus NetBox importieren

1. **Resources → Inventories → [Inventory] → Sources → Add**
2. Source: `NetBox`
3. NetBox-URL und Credentials eintragen
4. Sync starten

---

## Teil 9 — MCP-Server (KI-Integration, optional)

Der MCP-Server erlaubt KI-Assistenten (z.B. Claude) die Steuerung von awx-ng.

### 9.1 Token erstellen

1. **Resources → API Tokens → Create Token**
2. Scope: `write`
3. Token einmalig kopieren und sicher aufbewahren

### 9.2 MCP in Claude Code konfigurieren

```json
{
  "mcpServers": {
    "awx-ng": {
      "command": "curl",
      "args": ["-s", "-X", "POST", "http://localhost:8052/mcp"],
      "env": {
        "AWX_TOKEN": "<dein-token>"
      }
    }
  }
}
```

### 9.3 Verfügbare MCP-Tools

| Tool | Beschreibung |
|------|-------------|
| `awx_list_projects` | Projekte auflisten |
| `awx_list_inventories` | Inventories auflisten |
| `awx_run_playbook` | Playbook starten |
| `awx_list_project_files` | Dateibaum eines Projekts |
| `awx_read_project_file` | Dateiinhalt lesen |
| `awx_write_project_file` | Datei schreiben |
| `awx_sync_project` | Git-Sync auslösen |
| `awx_get_project_roles` | Rollen auflisten |
| `awx_get_role_variables` | Rollen-Variablen lesen |

---

## Zusammenfassung: Minimaler Workflow

```
1. .env befüllen (ANSIBLE_REPO_PATH + AWX_ADMIN_PASSWORD)
2. docker compose build && docker compose up -d
3. UI: Organisation anlegen
4. UI: Inventory anlegen
5. UI: Editor → "+" → Mountpoint-Projekt anlegen (Verzeichnis: ansible03)
6. UI: Host anlegen + host_vars setzen
7. UI: Playbooks → "Create template" für das gewünschte Playbook
8. UI: Playbooks → "Run" → Limit eingeben → Run
9. UI: Views → Jobs → Output beobachten
```
