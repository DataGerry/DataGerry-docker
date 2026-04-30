# DataGerry – Docker Setup

Dieses Repository stellt ein einfaches Docker-Compose-Setup für [DataGerry](https://datagerry.com) bereit – eine Open-Source-CMDB (Configuration Management Database) für die Verwaltung von IT-Assets und Infrastruktur.

Das Setup startet drei Container:

| Service       | Image                              | Beschreibung                          |
|---------------|------------------------------------|---------------------------------------|
| `dg-frontend` | `becongmbh/datagerry-frontend`     | Nginx-basiertes Web-Frontend          |
| `dg-backend`  | `becongmbh/datagerry-backend`      | DataGerry API / Application Server    |
| `mongodb`     | `mongo:6.0.25`                     | MongoDB als Datenbank                 |

---

## Voraussetzungen

- [Docker](https://docs.docker.com/get-docker/) (≥ 20.10)
- [Docker Compose](https://docs.docker.com/compose/install/) (v2)
- Freie Ports: **80**, **443** (Frontend) und **4000** (Backend-API)

---

## Quick Start

```bash
# Repository klonen
git clone https://github.com/DataGerry/DataGerry-docker.git
cd DataGerry-docker

# Container starten
docker compose up -d
```

Anschließend ist DataGerry erreichbar unter:

- **Web-UI:** http://localhost
- **API:** http://localhost:4000

### Standard-Login

Beim ersten Start wird ein Admin-Benutzer angelegt. Die Zugangsdaten findest du in der offiziellen [DataGerry-Dokumentation](https://docs.datagerry.com).

---

## Verzeichnisstruktur

```
.
├── docker-compose.yml
└── conf/
    ├── nginx.conf           # Nginx-Konfiguration (HTTP)
    ├── nginx-ssl.conf       # Nginx-Konfiguration (HTTPS)
    ├── cmdb.conf            # DataGerry-Backend-Konfiguration
    └── ssl/
        ├── certs/           # SSL-Zertifikate (.crt)
        └── private/         # SSL-Schlüssel (.key)
```

---

## Konfiguration

### Backend (`conf/cmdb.conf`)

Die Datei `cmdb.conf` wird ins Backend gemountet (`/etc/datagerry/cmdb.conf`) und steuert das Verhalten der DataGerry-Instanz (Datenbank-Verbindung, Auth-Provider, Logging etc.).

> **Hinweis:** Der MongoDB-Host wird bereits per Environment-Variable `DATAGERRY_Database_host=dg-mongodb` im `docker-compose.yml` gesetzt und überschreibt entsprechende Werte aus der `cmdb.conf`.

### Frontend (`conf/nginx.conf`)

Steuert Routing, Proxy-Pass zum Backend und TLS-Konfiguration.

---

## SSL / HTTPS aktivieren

Standardmäßig läuft DataGerry über HTTP. Um HTTPS zu aktivieren:

1. **Zertifikate ablegen:**
   ```bash
   cp dein-zertifikat.crt conf/ssl/certs/
   cp dein-schluessel.key  conf/ssl/private/
   ```

2. **`docker-compose.yml` anpassen** – im Service `dg-frontend` (und optional `dg-backend`) die folgenden Zeilen umschalten:

   ```yaml
   volumes:
   # comment for ssl
   #  - ./conf/nginx.conf:/etc/nginx/conf.d/default.conf
   # uncomment for ssl
     - ./conf/nginx-ssl.conf:/etc/nginx/conf.d/default.conf
     - ./conf/ssl/certs/:/etc/ssl/certs/
     - ./conf/ssl/private/:/etc/ssl/private/
   ```

3. **`conf/nginx-ssl.conf`** an die Dateinamen deiner Zertifikate anpassen.

4. **Stack neu starten:**
   ```bash
   docker compose up -d --force-recreate dg-frontend
   ```

DataGerry ist anschließend unter `https://<dein-host>` erreichbar.

---

## Verwaltung

### Container-Status prüfen
```bash
docker compose ps
```

### Logs anzeigen
```bash
docker compose logs -f                  # alle Services
docker compose logs -f dg-backend       # nur Backend
```

### Stack stoppen
```bash
docker compose stop                     # Container stoppen (Daten bleiben)
docker compose down                     # Container entfernen (Volumes bleiben)
docker compose down -v                  # ⚠️ Container UND Volumes entfernen
```

### Auf neue Version aktualisieren
```bash
docker compose pull
docker compose up -d
```

---

## Backup & Restore

Die MongoDB-Daten liegen im benannten Volume `mongodb-data`.

### Backup erstellen
```bash
docker exec dg-mongodb mongodump \
  --archive=/data/db/datagerry-$(date +%F).archive \
  --gzip

docker cp dg-mongodb:/data/db/datagerry-$(date +%F).archive ./backups/
```

### Backup einspielen
```bash
docker cp ./backups/datagerry-2025-01-15.archive dg-mongodb:/tmp/

docker exec dg-mongodb mongorestore \
  --archive=/tmp/datagerry-2025-01-15.archive \
  --gzip --drop
```

> **Tipp:** Lege Backups regelmäßig per Cronjob an und sichere zusätzlich das Verzeichnis `conf/`.

---

## Troubleshooting

**Frontend zeigt „502 Bad Gateway"**
Das Backend ist (noch) nicht erreichbar. Prüfe mit `docker compose logs dg-backend`, ob der Start abgeschlossen ist – beim ersten Hochfahren kann das einige Sekunden dauern.

**Backend kann nicht zur MongoDB verbinden**
Stelle sicher, dass der Container `dg-mongodb` läuft (`docker compose ps`) und dass keine andere Anwendung den MongoDB-Port belegt. Prüfe ggf. die Logs mit `docker compose logs mongodb`.

**Port 80 / 443 bereits belegt**
Passe das Port-Mapping im Service `dg-frontend` an, z. B.:
```yaml
ports:
  - 8080:80
  - 8443:443
```

**Konfigurationsänderungen werden nicht übernommen**
Nach Änderungen in `conf/` muss der entsprechende Container neu gestartet werden:
```bash
docker compose restart dg-frontend
docker compose restart dg-backend
```

---

## Weiterführende Links

- 🌐 [DataGerry Website](https://datagerry.com)
- 📖 [Offizielle Dokumentation](https://docs.datagerry.com)
- 💻 [DataGerry auf GitHub](https://github.com/DataGerry/DataGerry)
- 🐳 [Docker Hub – becongmbh](https://hub.docker.com/u/becongmbh)

---

## Lizenz

DataGerry steht unter der [AGPL-3.0-Lizenz](https://www.gnu.org/licenses/agpl-3.0.html).
Dieses Docker-Setup wird im Rahmen desselben Projekts bereitgestellt.
