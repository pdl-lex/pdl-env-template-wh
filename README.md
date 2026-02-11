# Deployment-Anleitung: Python-Tools auf LexoTerm-Server

## Übersicht

Diese Anleitung beschreibt, wie Python-Tools (z.B. Reflex/Dash/Flask-Apps) auf den LexoTerm-Server deployet werden. Das Setup nutzt Docker und Caddy für Reverse-Proxy-Routing.

## Voraussetzungen

- Tool ist in einem Git-Repository (z.B. `https://github.com/pdl-lex/tool_x`)
- Tool nutzt `uv` für Package-Management (`pyproject.toml` + `uv.lock`)
- SSH-Zugang zum Server
- Zugriff auf 138.246.225.215 erfordert VPN (auch intern), VPN-Profil: Full-Tunnel
- Zugewiesene Subdomain (z.B. `wh.dev.lexoterm.de`)

## Verzeichnisstruktur auf dem Server

```
~/lt-tools/                          # Deployment-Root
├── Caddyfile                        # Routing-Konfiguration
├── compose.yml                      # Orchestriert alle Services
├── html/                            # Statische Test-Seite
│   └── index.html
├── pdl-ltlabtools-dictconsistency/  # Tool 1 (Git-Repo)
│   ├── Dockerfile
│   ├── rxconfig.py
│   └── ...
├── tool2/                           # Tool 2 (Git-Repo)
│   ├── Dockerfile
│   └── ...
└── tool3/                           # Tool 3 (Git-Repo)
    └── ...
```

## Workflow: Neues Tool deployen

### 1. Tool lokal vorbereiten

#### a) Dockerfile erstellen

Erstelle ein `Dockerfile` im Root des Tool-Repos:

**Für Reflex-Apps:**
```dockerfile
FROM python:3.13-slim

WORKDIR /app

# uv installieren
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

# System dependencies für Reflex
RUN apt-get update && apt-get install -y \
    nodejs \
    npm \
    unzip \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Dependencies kopieren
COPY pyproject.toml uv.lock* ./

# Dependencies installieren
RUN uv sync --frozen

# App-Code kopieren
COPY . .

# Reflex initialisieren
RUN uv run reflex init

# Ports exposen
EXPOSE 3000

# Production mode
ENV REFLEX_ENV=prod

# App starten
CMD ["uv", "run", "reflex", "run", "--env", "prod"]
```

**Für Dash/Flask-Apps:**
```dockerfile
FROM python:3.13-slim

WORKDIR /app

COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv

COPY pyproject.toml uv.lock* ./
RUN uv sync --frozen

COPY . .

EXPOSE 8000

CMD ["uv", "run", "python", "app.py"]
```

**Für React/TypeScript/Vite-Apps:**
```dockerfile
# Stage 1: Build
FROM node:20-slim AS builder

WORKDIR /app

# Package files kopieren
COPY package.json package-lock.json* ./

# Dependencies installieren
RUN npm ci

# Source code kopieren
COPY . .

# Production build
RUN npm run build

# Stage 2: Production
FROM caddy:2-alpine

# Build-Artefakte vom Builder kopieren
COPY --from=builder /app/dist /srv

# Caddyfile kopieren
COPY Caddyfile /etc/caddy/Caddyfile

EXPOSE 80

CMD ["caddy", "run", "--config", "/etc/caddy/Caddyfile"]
```

**Caddyfile für die React-App** (im Tool-Root erstellen):
```caddyfile
:80 {
    root * /srv
    encode gzip
    try_files {path} /index.html
    file_server
}
```


#### b) Reverse-Proxy-Config (nur für Reflex)

Wenn deine App unter einem Pfad-Prefix laufen soll (z.B. `/toolname`), passe `rxconfig.py` an:

```python
import reflex as rx

config = rx.Config(
    app_name="dein_tool_name",
    backend_host="0.0.0.0",
    frontend_path="/toolname",  # Pfad-Prefix
    plugins=[...],
)
```

#### c) Git committen & pushen

```bash
git add Dockerfile rxconfig.py  # bzw. relevante Dateien
git commit -m "Add Docker deployment config"
git push
```

### 2. Auf dem Server: Tool klonen

```bash
ssh [user]@[server]
cd ~/lt-tools

# Tool-Repository klonen
git clone https://github.com/pdl-lex/dein-tool.git
cd dein-tool

# Prüfen, dass Dockerfile vorhanden ist
ls -la Dockerfile
```

### 3. compose.yml erweitern

```bash
cd ~/lt-tools
nano compose.yml
```

Füge einen neuen Service hinzu:

```yaml
services:
  caddy:
    # ... bleibt unverändert ...

  dictconsistency:
    # ... bestehende Services bleiben ...

  dein-tool:  # Neuer Service
    build: ./dein-tool
    ports:
      - "3002:3000"  # Externer Port:Interner Port (Port erhöhen!)
    restart: unless-stopped
    networks:
      - dev-network
    # Optional: Volumes für persistente Daten
    # volumes:
    #   - ./dein-tool-data:/app/data

networks:
  dev-network:
    driver: bridge

volumes:
  caddy_data:
  caddy_config:
```

**Wichtig:** 
- Jeder Service braucht einen **eigenen externen Port** (3001, 3002, 3003, ...)
- Service-Name muss eindeutig sein
- Interner Port hängt von der App ab (oft 3000, 8000, 5000)

### 4. Caddyfile erweitern

```bash
nano Caddyfile
```

Füge Routing für das neue Tool hinzu:

```
:80 {
    # Bestehende Tools
    redir /dictconsistency /dictconsistency/ permanent
    handle /dictconsistency/* {
        reverse_proxy dictconsistency:3000
    }
    
    # NEUES TOOL
    redir /toolname /toolname/ permanent
    handle /toolname/* {
        reverse_proxy dein-tool:3000  # Service-Name:Interner-Port
    }
    
    # Default: Testseite
    handle {
        root * /srv
        file_server
        try_files {path} /index.html
    }
}
```

**Alternative: Eigene Subdomain (ohne Pfad-Prefix)**

Falls dein Tool eine eigene Subdomain bekommt (z.B. `tool.dev.lexoterm.de`):

1. **Kollege muss im zentralen Caddyfile eintragen:**
   ```
   tool.dev.lexoterm.de {
       reverse_proxy host.docker.internal:8082  # Neuer Port!
   }
   ```

2. **Dein lokales Caddyfile wird einfacher:**
   ```
   :80 {
       reverse_proxy dein-tool:3000
   }
   ```

3. **compose.yml Port anpassen:**
   ```yaml
   caddy:
     ports:
       - "8082:80"  # Muss mit zentralem Caddyfile übereinstimmen!
   ```

### 5. Deployment ausführen

```bash
cd ~/lt-tools

# Container bauen und starten
docker compose build dein-tool
docker compose up -d

# Logs prüfen
docker compose logs dein-tool --tail=50

# Alle laufenden Container checken
docker compose ps
```

### 6. Testen

**Lokal (auf dem Server):**
```bash
curl http://localhost:8080/toolname/
```

**Im Browser:**
```
https://wh.dev.lexoterm.de/toolname/
```

## Updates deployen

### Workflow nach Code-Änderungen

```bash
# 1. Lokal entwickeln
git add .
git commit -m "Feature XYZ"
git push

# 2. Auf dem Server
ssh [user]@[server]
cd ~/lt-tools/dein-tool
git pull

# 3. Container neu bauen und starten
cd ~/lt-tools
docker compose build dein-tool
docker compose up -d

# 4. Logs checken
docker compose logs dein-tool --tail=20
```

### Nur Caddyfile/compose.yml geändert

```bash
cd ~/lt-tools
# Änderungen machen
nano Caddyfile

# Caddy neu laden (ohne rebuild)
docker compose restart caddy
```

## Troubleshooting

### App läuft nicht

```bash
# Container-Status prüfen
docker compose ps

# Logs ansehen
docker compose logs dein-tool --tail=100

# Container neu starten
docker compose restart dein-tool

# Kompletter Neustart
docker compose down
docker compose up -d
```

### 502 Bad Gateway

- **Läuft der Container?** `docker compose ps`
- **Läuft die App im Container?** `docker compose logs dein-tool`
- **Ist der Port richtig?** Prüfe `compose.yml` und `Caddyfile`
- **Netzwerk OK?** `docker exec lt-tools-caddy-1 wget -O- http://dein-tool:3000`

### 404 Not Found

- **Caddyfile-Routing falsch?** Prüfe Handle-Blöcke
- **Pfad-Prefix stimmt?** Bei Reflex: `frontend_path` in `rxconfig.py`
- **Trailing Slash?** Redirect eingebaut? (`redir /tool /tool/`)

### Port bereits belegt

```bash
# Welche Ports sind belegt?
docker compose ps

# Freien Port in compose.yml wählen
# Ports: 3001, 3002, 3003, 3004, ...
```

## Cheat Sheet

```bash
# Container-Status
docker compose ps

# Logs ansehen (live)
docker compose logs -f dein-tool

# Service neu bauen
docker compose build dein-tool

# Service neu starten
docker compose restart dein-tool

# Alles stoppen
docker compose down

# Alles starten
docker compose up -d

# In Container einsteigen (Debug)
docker exec -it lt-tools-dein-tool-1 /bin/bash

# Netzwerk inspizieren
docker network inspect lt-tools_dev-network
```

## Wichtige Dateien

| Datei | Zweck | Wann ändern? |
|-------|-------|--------------|
| `~/lt-tools/compose.yml` | Orchestriert alle Services | Neues Tool, Port-Änderung |
| `~/lt-tools/Caddyfile` | Routing-Regeln | Neues Tool, Pfad-Änderung |
| `~/lt-tools/[tool]/Dockerfile` | Container-Build | Dependencies, Setup |
| `~/lt-tools/[tool]/rxconfig.py` | Reflex-Config | Pfad-Prefix, API-URL |

## Port-Übersicht

| Service | Externer Port | Interner Port | URL |
|---------|---------------|---------------|-----|
| Haupt-Caddy | 8080 | 80 | - |
| dictconsistency | 3001 | 3000 | `/dictconsistency` |
| tool2 | 3002 | 3000 | `/tool2` |
| tool3 | 3003 | 8000 | `/tool3` |

## Nächste Schritte

- [ ] Template-Fork auf GitHub umbenennen in `wh-lexotools-deployment`
- [ ] CI/CD mit GitHub Actions einrichten (optional)
- [ ] Monitoring/Logging-Setup (optional)
- [ ] Backup-Strategie für persistente Daten

---

**Erstellt:** 2026-02-10  
**Server:** ba00000-lex.srv.mwn.de  
**Domain:** wh.dev.lexoterm.de