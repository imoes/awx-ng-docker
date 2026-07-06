# awx-ng

AWX-Fork mit Foreman-artiger Variablenverwaltung für Ansible-Automatisierung.

Basiert auf [AWX 24.6.1](https://github.com/ansible/awx) (Apache 2.0).

## Was ist anders als upstream AWX?

| Feature | AWX upstream | awx-ng |
|---|---|---|
| Rollen-Variablen-Extraktion | ✗ | ✓ Automatischer Scan nach jedem git sync |
| Per-Host aggregierte Variablen | ✗ | ✓ Zusammengeführte Ansicht: Rollen-Defaults → Gruppen-Vars → Host-Vars |
| Survey-Generierung aus Rollen | ✗ | ✓ Ein Klick → Survey-Spec aus Rollen-Defaults |
| Playbook-Editor | ✗ | ✓ Monaco-basierter Datei-Editor mit YAML-Linting |
| Git-Integration im Editor | ✗ | ✓ Commit & Push direkt aus der UI |
| Standort-/Site-Management | ✗ | ✓ Sites + SSH-Credentials + Umgebungsvariablen pro Site |
| Runner-Verwaltung | ✗ | ✓ Remote Execution Nodes registrieren und Sites zuweisen |
| MCP-Server | ✗ | ✓ JSON-RPC 2.0 Endpoint für KI-Assistenten |
| Root-Passwort-Hashing | ✗ | ✓ sha512crypt, gespeichert in Host-Vars |
| NetBox-Integration | ✗ | ✓ Location-Reconcile, Inventory-Source-bereit |
| API-Token-Verwaltung | ✗ | ✓ Persönliche OAuth2-Tokens (UI + API) |
| Ansible Vault Store | ✗ | ✓ Benannte Vaults, automatisch generierte Passwörter, auto-injiziert beim Job-Start |
| SSO / Keycloak | optional | ✓ OIDC vorkonfiguriert über Umgebungsvariablen |

Vollständige Schritt-für-Schritt-Dokumentation: **[WORKFLOW-de.md](WORKFLOW-de.md)**  
Remote-Runner-Einrichtung: **[PROXIES-de.md](PROXIES-de.md)**

## Schnellstart

```bash
git clone https://github.com/imoes/awx-ng-docker.git
cd awx-ng-docker

# 1. Secrets generieren
mkdir -p secrets
python3 -c "import secrets; print(secrets.token_hex(32))" > secrets/secret_key
python3 -c "import secrets; print(secrets.token_hex(16))" > secrets/pg_password
chmod 600 secrets/*

# 2. Konfiguration anlegen
cp .env.example .env
$EDITOR .env   # mindestens AWX_ADMIN_PASSWORD und ANSIBLE_REPO_PATH setzen

# 3. Daten-Verzeichnisse anlegen
mkdir -p data/postgres data/redis data/projects
mkdir -p data/receptor && chmod 777 data/receptor

# 4. Bauen und starten
docker compose build
docker compose up -d

# 5. Status prüfen (Web-UI startet nach ~60s)
docker compose ps
```

Web-UI: **http://localhost:8052** — Login: `admin` / Passwort aus `.env`

## Voraussetzungen

- Docker ≥ 24
- Docker Compose v2
- 4 GB RAM, 20 GB freier Platz
- Ein Ansible-Repository auf dem Host (Pfad in `ANSIBLE_REPO_PATH`)

## Architektur

```
Browser / API-Client / MCP-Client
        │
        ▼
  ┌─────────────┐
  │  awx_web    │  nginx + uWSGI · Port 8052 · REST API + Web-UI + MCP-Endpoint
  └──────┬──────┘
         │ Job-Dispatch
         ▼
  ┌─────────────┐
  │  awx_task   │  Dispatcher · Rollen-Scan-Hook · Credential-Injektion
  └──────┬──────┘
         │ Receptor-Socket / Mesh
         ▼
  ┌─────────────┐   ┌─────────────────────┐
  │   awx_ee    │   │  Remote Runner       │  optional, verbindet sich ausgehend
  │  (lokal)    │   │  (ansible-runner.    │  auf Port 2222 (NAT-freundlich)
  └─────────────┘   │   site.example.com)  │
                    └─────────────────────┘
```

`AWX_DISABLE_CONTAINER_ISOLATION=True` — Jobs laufen direkt im Container, kein nested Docker nötig.

## Konfiguration

### .env

| Variable | Pflicht | Beschreibung |
|---|---|---|
| `AWX_ADMIN_PASSWORD` | ✓ | Admin-Passwort für die Web-UI |
| `ANSIBLE_REPO_PATH` | ✓ | Host-Pfad zum Ansible-Repository (wird als `/var/lib/awx/projects/ansible03` eingehängt) |
| `NETBOX_URL` | — | NetBox-URL für Location-Reconcile |
| `NETBOX_TOKEN` | — | NetBox API-Token |
| `OIDC_ENDPOINT` | — | Keycloak-Endpoint (`https://.../auth/realms/<realm>`) |
| `OIDC_KEY` | — | Keycloak Client-ID |
| `OIDC_SECRET` | — | Keycloak Client-Secret |
| `OIDC_VERIFY_SSL` | — | SSL-Verifikation (Standard: `true`) |

### config/custom.py

Django-Settings-Overlay — wird als Bind-Mount eingehängt, kein Rebuild nötig.
Enthält u.a. `INSTALLED_APPS`, `NETBOX_URL`, `AWX_DISABLE_CONTAINER_ISOLATION`.

### Ansible-Repository-Einbindung

Das Ansible-Repository (`ANSIBLE_REPO_PATH`) wird in alle drei Container eingehängt:

```yaml
- ${ANSIBLE_REPO_PATH}:/var/lib/awx/projects/ansible03:rw
```

AWX-Projekt anlegen: SCM Type **Manual**, Playbook Directory `ansible03`.

## Benutzeroberflächne (awx-ng)

Alle awx-ng-Screens sind in die bestehende AWX-Navigation integriert.

### Resources

| Screen | Pfad | Beschreibung |
|--------|------|-------------|
| Roles | `/roles` | Rollen-Übersicht mit Variablen, Tags, Handlern; manueller Scan |
| Playbooks | `/playbooks` | Playbook-Liste mit Play-Details (Hosts, Rollen, Tags); Job starten |
| Editor | `/editor` | Datei-Editor für Playbooks und Rollen (Monaco + YAML-Linting + Git) |
| Vaults | `/vaults` | Ansible-Vault-Stores verwalten (Key-Value-Paare → verschlüsselte YAML-Datei); Passwörter werden automatisch beim Job-Start injiziert |
| Playbook Builder | `/playbook-builder` | Visueller (Blockly) Drag&Drop-Playbook-Editor — siehe unten |

### Administration

| Screen | Pfad | Beschreibung |
|--------|------|-------------|
| Runners | `/runner_sites` | Execution Nodes registrieren, Sites zuweisen, Health-Checks |
| Sites | `/locations` | Standorte verwalten (= AWX Instance Groups); SSH-Credential + Env + ansible.cfg pro Site |
| API Tokens | `/tokens` | Persönliche OAuth2-Tokens für REST API und MCP |

### Editor-Features

- **Datei-Baum** mit Lazy-Loading und Kontextmenü (umbenennen, duplizieren, löschen, neu)
- **Monaco YAML-Editor** mit Jinja2-Syntax-Highlighting
- **YAML-Linting** in Echtzeit (800ms Debounce) — Fehler inline markiert
- **Datei-Upload** (ZIP/tar.gz werden automatisch entpackt)
- **Mountpoint-Projekt anlegen** (`+`-Button): bindet ein vorhandenes Verzeichnis als AWX-Projekt ein
- **Git-Panel** (nur für Git-Repos): Branch-Anzeige, Dirty-/Ahead-Indikatoren, Commit mit Message, Push

### Sites & Runner

Sites sind 1:1 mit AWX Instance Groups verknüpft — beim Anlegen einer Site wird die Instance Group automatisch erstellt (und beim Löschen entfernt). System-Gruppen (`controlplane`, `default`) sind geschützt.

**Auflösung zur Laufzeit:** AWX wählt einen Runner aus der Site → per-Runner-Override gewinnt, sonst Site-Default für SSH-Credential, Umgebungsvariablen und ansible.cfg.

### Playbook Builder (visuell/Blockly)

Ein Drag&Drop-Playbook-Editor (`/playbook-builder`), ähnlich im Ansatz wie der Blockly-Editor von
ioBroker: Plays werden aus Blöcken zusammengesteckt — Module, Rollen, Tasks — statt YAML von Hand
zu schreiben.

- **Modul-Katalog**: alle 71 `ansible.builtin`-Module, automatisch aus `ansible-doc -j` generiert
  und als JSON committed (typisierte Felder: choices → Dropdown, bool → Checkbox, sonst Text).
  Blöcke zeigen nur **Pflicht- (mit `*`) + Primär-Parameter**; der Rest wird bei Bedarf über ein
  „add parameter…"-Dropdown ergänzt — so bleiben die Blöcke klein und leicht verbindbar.
- **Live-Suche** filtert Module/Rollen beim Tippen; **New playbook** / **New role** starten ein
  leeres Dokument.
- **Roles-Kategorie**: pro Projekt befüllt aus `GET /api/v2/projects/{id}/roles/` — aktualisiert
  sich beim Projektwechsel.
- **Live-YAML-Vorschau** aktualisiert sich beim Bauen; **Lint & Save** prüft mit denselben
  YAML-/ansible-lint-Checks wie der Datei-Editor, bevor gespeichert wird.
- **Öffnen-Dialog** (*Open playbook…* / *Open role…*): ein *bestehendes* Playbook oder eine Rolle
  auswählen und es wird als Blöcke rekonstruiert — erkannte Module werden zu typisierten Blöcken
  (inklusive dokumentierter ansible-doc-Parameter-Aliase, z.B. `dest:`/`name:` statt `path:` bei
  `file`), Rollen zu Rollen-Blöcken, alles andere (Module aus anderen Collections, `block:`/
  `rescue:`, sonstige Play-Level-Keys) bleibt verlustfrei in einem Raw-Fallback-Block bzw. -Feld
  erhalten.
- **Rollen-Bearbeitung (Reiter Tasks/Handlers/Defaults/Vars)**: eine Rolle besteht aus 4 Dateien
  (`roles/<name>/{tasks,handlers,defaults,vars}/main.yml`), bearbeitet in **einer**
  Blockly-Sitzung über Reiter. *New role* legt alle 4 an (auch unberührte, als leere Datei) und
  *Lint & Save* schreibt alle 4 gleichzeitig — kein manuelles Anlegen von Verzeichnissen nötig.
  *Open role…* lädt alle 4 Dateien vorab, Reiter-Wechsel ist danach sofort.
- **Task-Einstellungen als eigene Blöcke**: `when`/`tags`/`notify`/`register`/`loop`/
  `delegate_to`/`become`/`ignore_errors` sind eigenständige, verkettbare Ein-Zeilen-Blöcke (keine
  Felder auf dem Modul-Block mehr) — so wird eine `when:`-Bedingung sauber eingebettet dargestellt
  statt seitlich angedockt.
- **Bedingungen (`when:`) aus Blöcken bauen**: Vergleiche (`==`/`!=`/`>`/`<`/`in`/…), Jinja-„is"-Tests
  (`defined`/`changed`/`failed`/…), `and`/`or`/`not` — visuell zusammengesteckt statt Jinja zu
  tippen. Eine bestehende `when:`-Bedingung wird beim Import automatisch in diese Blöcke zerlegt;
  alles außerhalb der unterstützten Grammatik (Filter, Funktionsaufrufe) bleibt als Text erhalten.
- **Variablen als Blockly-Elemente**: neue Variablen direkt über das „+ Add variable"-Formular im
  Variablen-Panel anlegen (Name + optionaler Wert), oder per `var`-Block (Play-`vars:`, oder über
  den Defaults/Vars-Reiter einer Rolle) — beides erscheint sofort in der Liste. Das Panel listet
  außerdem Rollen-/Vault-Variablen sowie ~58 kuratierte `ansible_facts` und ~13 „Magic Variables"
  (`inventory_hostname`, `group_names`, `hostvars`, …) — auf ein Textfeld gezogen ergibt
  `{{ name }}`, auf die leere Canvas gezogen einen wiederverwendbaren Variablen-Block.
- **Globale Live-Suche**: eine „🔍 Search"-Toolbox-Rubrik durchsucht alle Blöcke aus allen
  Kategorien (Module, Rollen, Bedingungen, Task-Einstellungen, …) in einem gemeinsamen Flyout.
- **Layout-Persistenz**: die visuelle Blockanordnung wird als `<name>.blockly.json`-Sidecar neben
  jeder Datei gespeichert — beim erneuten Öffnen erscheint exakt dieselbe Canvas (statt die YAML
  komplett neu zu parsen).
- **3D-Blockstil** (geras-Renderer + Classic-Theme) ähnlich dem Blockly-Editor von ioBroker.
- **5. Reiter „Templates"** in der Rollen-Bearbeitung: verwaltet `roles/<name>/templates/*.j2`
  (die Jinja2-Quelldateien des `template`-Moduls) — Dateiliste + Text-/Monaco-Editor direkt neben
  Tasks/Handlers/Defaults/Vars, kein Screen-Wechsel zum generischen Datei-Editor nötig. Anlegen,
  Bearbeiten, Speichern (sofort pro Datei, unabhängig vom großen „Lint & Save") und Löschen.
  Templates brauchen keine `.j2`-Endung — auch endungslose Dateien (wie sie viele reale Rollen
  bereits haben) lassen sich öffnen und bearbeiten.

Fast reines Frontend-Feature — ein kleiner Backend-Fix war nötig: der Datei-Editor-Endpunkt
erlaubte bisher nur eine feste Liste von Dateiendungen (`.yml`/`.yaml`/`.j2`/`.jinja2`/`.conf`/
`.ini`/`.md`/`.txt`/`.cfg`/`.json`) und blockierte damit reale, endungslose Rollen-Templates.
Dateien unterhalb von `roles/*/templates/` oder `roles/*/files/` sind jetzt unabhängig von der
Endung erlaubt (Größenlimit von 512 KB und UTF-8-Prüfung bleiben der eigentliche Schutz);
ansonsten nutzt das Feature die unten dokumentierten Datei-, Rollen- und Vault-Endpunkte weiter.

## Custom API-Endpoints

Basis: `http://localhost:8052/api/v2/`

### Rollen-Variablen

```
GET  /api/v2/projects/{id}/role_variables/               # Variablen aus defaults/ + vars/
GET  /api/v2/projects/{id}/role_variables/?role_name=img_docker
POST /api/v2/projects/{id}/role_variables/scan/trigger/  # manueller Scan
GET  /api/v2/projects/{id}/role_tags/                    # Tags aus tasks/**/*.yml
GET  /api/v2/projects/{id}/role_handlers/                # Handlers aus handlers/main.yml
GET  /api/v2/projects/{id}/roles/                        # Alle Rollen (Disk + DB)
```

### Datei-Editor

```
GET    /api/v2/projects/{id}/files/                      # Verzeichnis-Listing
GET    /api/v2/projects/{id}/files/content/?path=...     # Dateiinhalt lesen
PUT    /api/v2/projects/{id}/files/content/?path=...     # Datei schreiben (+ git commit)
DELETE /api/v2/projects/{id}/files/content/?path=...     # Datei löschen
POST   /api/v2/projects/{id}/files/rename/               # Datei umbenennen (git mv)
POST   /api/v2/projects/{id}/files/upload/               # Datei/Archiv hochladen
POST   /api/v2/projects/{id}/files/lint/                 # YAML-Validierung
```

### Git-Operationen

```
GET  /api/v2/projects/{id}/git/                          # Status, Branch, Log, Ahead-Count
POST /api/v2/projects/{id}/git/  {"action": "commit", "message": "..."}
POST /api/v2/projects/{id}/git/  {"action": "push"}
```

### Playbook-Metadaten

```
GET  /api/v2/projects/{id}/plays/                        # Playbook-Liste mit Play-Details
GET  /api/v2/projects/{id}/plays/?playbook=site.yml      # Einzelnes Playbook
POST /api/v2/projects/{id}/launch/                       # Job starten (jt_id + limit + location_id)
GET  /api/v2/projects/{id}/variable_usages/?role=...&var=...  # Variablen-Verwendung
```

### Host-Variablen

```
GET    /api/v2/hosts/{id}/aggregated_variables/          # Zusammengeführter Variablen-Stack
GET    /api/v2/hosts/{id}/role_variables/                # Host-Variablen
PATCH  /api/v2/hosts/{id}/role_variables/{var_name}/     # Wert überschreiben
DELETE /api/v2/hosts/{id}/role_variables/{var_name}/     # Auf Default zurücksetzen
POST   /api/v2/hosts/{id}/assign_roles/                  # Rollen dem Host zuweisen
POST   /api/v2/hosts/{id}/run/                           # Job für diesen Host starten
POST   /api/v2/hosts/{id}/set_root_password/             # Root-Passwort hashen + speichern
```

### Standorte & Runner

```
GET/POST   /api/v2/locations/                            # Sites auflisten / anlegen
PATCH/DEL  /api/v2/locations/{id}/                       # Site bearbeiten / löschen
POST       /api/v2/locations/reconcile/                  # Abgleich mit NetBox
GET/POST   /api/v2/execution_node_locations/             # Runner-Site-Zuordnungen
PATCH/DEL  /api/v2/execution_node_locations/{id}/
POST       /api/v2/runners/register/                     # Runner registrieren
POST       /api/v2/runners/deprovision/                  # Runner entfernen
```

### Survey & Tools

```
POST /api/v2/job_templates/{id}/generate_survey/         # Survey aus Rollen-Defaults
POST /api/v2/tools/hash_password/                        # sha512crypt-Hash
```

### Ansible Vault Store

```
GET    /api/v2/vaults/                                   # Alle Vaults auflisten (ohne Passwort/Variablen)
POST   /api/v2/vaults/                                   # Vault anlegen (generiert Passwort + AWX-Credential)
GET    /api/v2/vaults/{id}/                              # Vault mit Klartext-Variablen abrufen
PATCH  /api/v2/vaults/{id}/                              # Variablen oder Beschreibung ändern
DELETE /api/v2/vaults/{id}/                              # Vault + AWX-Credential löschen
POST   /api/v2/vaults/{id}/generate/                     # Verschlüsselte Vault-Datei generieren
                                                         # Body: {"project_id": N} — schreibt in Projekt
```

Das Vault-Passwort wird **nie** über die API zurückgegeben. Es liegt in der DB und wird beim
Job-Start automatisch als AWX Vault-Credential injiziert. Im Playbook:

```yaml
vars_files:
  - vault-meine-vault.yml   # Datei wird per POST /api/v2/vaults/{id}/generate/ erstellt
```

### Tokens & MCP

```
GET/POST   /api/v2/tokens/                               # Persönliche OAuth2-Tokens
DELETE     /api/v2/tokens/{id}/
GET/POST   /mcp                                          # MCP-Server (JSON-RPC 2.0, Bearer-Auth)
```

## Rebuild nach Code-Änderungen

Custom-Code (`custom/`) ist ins Docker-Image gebaut — bei Änderungen:

```bash
docker compose build awx_web awx_task
docker compose up -d awx_web awx_task
```

Config-Dateien (`config/`) sind Bind-Mounts — reicht `docker compose restart awx_web`.

Bei Migrations-Änderungen:
```bash
docker compose run --rm awx_init awx-manage migrate customvars
```

## SSO / Keycloak

Keycloak-Setup (vom Keycloak-Admin):
1. Client `awx-ng` anlegen (confidential)
2. Redirect URI: `http://<host>:8052/sso/complete/oidc/`
3. Client-ID und Secret in `.env` eintragen (`OIDC_KEY`, `OIDC_SECRET`, `OIDC_ENDPOINT`)

## MCP-Server

Der MCP-Server (Model Context Protocol) erlaubt KI-Assistenten die direkte Steuerung von AWX.

- Endpoint: `https://<host>/mcp` (JSON-RPC 2.0)
- Auth: Bearer-Token (OAuth2) oder Django-Session
- Token erstellen: **Resources → API Tokens** oder `POST /api/v2/tokens/`
- Verfügbare Tools: `awx_run_playbook`, `awx_list_inventories`, `awx_list_projects`,
  `awx_list_project_files`, `awx_read_project_file`, `awx_write_project_file`, u.a.

### Prose-Authoring-Schicht (Klartext-Anfrage → Playbook)

14 MCP-Tools, die einem KI-Client genug *strukturierten, abrufbaren* Kontext geben, um korrektes
Ansible zu bauen — so ausgelegt, dass ein **kleines lokales Modell (7B, Ollama/vLLM/llama.cpp)**
ausreicht: nicht Kontext ersetzt Modellgröße, sondern die Aufgabe wird verkleinert (RAG statt
Recall, Schema-constrained Tool-I/O, Lint→Retry, `--check`-Dry-Run als Netz — Idempotenz/`--check`
sind hier *echt*, da echtes ansible-runner läuft).

- **Kontext**: `ansible_conventions` (cachebarer Best-Practice-Text), `get_catalog` (Kompakt-Digest
  aller 71 `ansible.builtin`-Module), `search_modules`/`search_roles`/`search_playbooks` (semantisch
  via bge-m3, lexikalischer Fallback), `get_module` (voller typisierter Param-Spec),
  `list_roles`/`get_role` (Projekt-Rollen als Bausteine).
- **Authoring-Schleife**: `draft_playbook` (schreiben + linten), `lint_playbook` (strukturierte
  Fehler für Retry), `check_run`/`check_result` (Check-Mode-Dry-Run + PLAY-RECAP-Auswertung =
  Idempotenz-Check).
- **Generation-Cache**: `cache_lookup`/`cache_store` — ein validiertes Ergebnis für dieselbe oder
  eine nahezu identische (Cosine ≥ 0.85) Anfrage wird mit null LLM-Tokens wiederverwendet.

Embeddings liegen als JSON + Cosine-in-Python (kein pgvector). `AWX_EMBED_URL` auf den
OpenAI-kompatiblen bge-m3-Endpoint setzen (Default `https://llamacpp03.ippen.media/embed`). **Nach
dem Deploy (und nach Projekt-Syncs) `docker compose exec awx_web awx-manage mcp_reindex` ausführen**,
um den semantischen Index zu bauen — bis dahin greift die lexikalische Suche.

## Projektstruktur

```
awx-ng-docker/
├── Dockerfile              # AWX 24.6.1 Basis + awx-ng Patches
├── Dockerfile.ee           # Execution Environment (ansible-runner + Receptor + Collections)
├── Dockerfile.proxy        # Remote Runner Image (identisch zu Dockerfile.ee)
├── docker-compose.yml      # Dreiergespann: awx_web + awx_task + awx_ee
├── docker-compose.proxy.yml # Remote Runner Compose (für externe Hosts)
├── .env.example            # Konfigurationsvorlage
├── config/
│   ├── custom.py           # Django-Settings-Overlay (Bind-Mount, kein Rebuild)
│   └── nginx_awx.conf      # nginx-Konfiguration (Bind-Mount, kein Rebuild)
├── scripts/
│   ├── init_awx.sh                  # DB-Migration + Admin-User beim Start
│   ├── setup-proxy.sh               # receptor.conf Generator für Remote Runner
│   └── launch_awx_task_ng.sh        # Task-Container-Entrypoint
├── receptor/
│   ├── receptor-control.conf        # Receptor Control Node (Port 2222)
│   └── receptor-proxy.conf.template # Remote Runner Template
├── secrets/                # secret_key + pg_password (nicht in Git)
├── data/                   # Laufzeit-Daten (nicht in Git)
└── custom/awx/
    ├── customvars/         # Django-App: Models, API, Migrations, MCP-Tools
    ├── api/urls/           # Gepatchte URL-Routen
    └── main/tasks/         # Gepatchte jobs.py (Credential-Injektion, Env-Vars)
```

## Lizenz

Apache License 2.0 — Volltext in [LICENSE](LICENSE), Attributionen in [NOTICE](NOTICE).

awx-ng ist ein Fork von [ansible/awx](https://github.com/ansible/awx)
(Copyright Ansible, a Red Hat Company), lizenziert unter Apache 2.0.
Alle Modifikationen stehen ebenfalls unter Apache 2.0.
