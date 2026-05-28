Markdown
# MyLab – Private Cloud & Security Infrastructure

Dieses Repository dokumentiert den Aufbau und die Konfiguration meiner privaten Server-Infrastruktur auf Basis von Ubuntu Server, Docker und Docker Compose.

## Architektur-Übersicht
Die Infrastruktur ist so konzipiert, dass alle Dienste isoliert in Docker-Containern laufen und über einen zentralen Einstiegspunkt abgesichert sowie geroutet werden.

* **Reverse Proxy:** Nginx Proxy Manager (Zentraler Pförtner, der den externen Traffic auf Port 80/443 kontrolliert und an die internen Container-Ports weiterleitet).
* **SSL-Verschlüsselung:** Automatisierte, dynamische Zertifikatsverwaltung via Let's Encrypt mit erzwungenem HTTPS (Force SSL) und HTTP/2-Unterstützung.
* **Passwort-Management:** Vaultwarden (Bitwarden-API in Rust) zur hochsicheren, ressourcenschonenden Verwaltung von Anmeldedaten.
* **Cloud-Speicher:** Nextcloud (Zentrale Datensynchronisation über eine ressourcenschonende, integrierte SQLite-Datenbank).

---

## Docker Compose Konfigurationen

### 1. Reverse Proxy (Nginx Proxy Manager)
Der Proxy fängt alle Anfragen der Subdomains ab und verteilt sie intern weiter.

```yaml
version: '3.8'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
Markdown
# MyLab – Private Cloud & Security Infrastructure

Dieses Repository dokumentiert den Aufbau und die Konfiguration meiner privaten Server-Infrastruktur auf Basis von Ubuntu Server, Docker und Docker Compose.

## Architektur-Übersicht
Die Infrastruktur ist so konzipiert, dass alle Dienste isoliert in Docker-Containern laufen und über einen zentralen Einstiegspunkt abgesichert sowie geroutet werden.

* **Reverse Proxy:** Nginx Proxy Manager (Zentraler Pförtner, der den externen Traffic auf Port 80/443 kontrolliert und an die internen Container-Ports weiterleitet).
* **SSL-Verschlüsselung:** Automatisierte, dynamische Zertifikatsverwaltung via Let's Encrypt mit erzwungenem HTTPS (Force SSL) und HTTP/2-Unterstützung.
* **Passwort-Management:** Vaultwarden (Bitwarden-API in Rust) zur hochsicheren, ressourcenschonenden Verwaltung von Anmeldedaten.
* **Cloud-Speicher:** Nextcloud (Zentrale Datensynchronisation über eine ressourcenschonende, integrierte SQLite-Datenbank).

---

## Docker Compose Konfigurationen

### 1. Reverse Proxy (Nginx Proxy Manager)
Der Proxy fängt alle Anfragen der Subdomains ab und verteilt sie intern weiter.

```yaml
version: '3.8'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
```

Markdown
### 2. Passwort-Tresor (Vaultwarden)
Sichere Passwort-Instanz, die intern auf Port `8080` lauscht und vom Proxy per SSL geschützt wird.

```yaml
version: '3.8'
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      - WEBSOCKET_ENABLED=true
    volumes:
      - ./vw-data:/data
    ports:
      - '8080:80'
```

### 3. Cloud-Zentrale (Nextcloud)
Die Nextcloud-Anwendung läuft in einem Container und nutzt für maximale Effizienz im die integrierte SQLite-Datenbankstruktur.

```yaml
version: '3.8'
services:
  nextcloud:
    image: nextcloud:latest
    container_name: nextcloud-app
    restart: always
    ports:
      - "8090:80"
    volumes:
      - ./nextcloud_data:/var/www/html
```


### 3. Backup einrichten 
Um vor Datencrashes sicher zu sein ist ein backup notwendig


```bash
#!/bin/bash

# Das Skript muss als root/sudo ausgeführt werden
if [ "$EUID" -ne 0 ]; then
  echo "Bitte starte das Skript mit: sudo $0"
  exit 1
fi

# Variablen definieren (Backup-Ordner außerhalb von SOURCE_DIR, um tar-Fehler zu vermeiden)
BACKUP_DIR="/home/$SUDO_USER/docker_backups"
SOURCE_DIR="/home/$SUDO_USER/docker"
TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")
BACKUP_NAME="docker_backup_$TIMESTAMP.tar.gz"

# Erstelle den Backup-Ordner, falls er noch nicht existiert
mkdir -p "$BACKUP_DIR"
chown "$SUDO_USER:$SUDO_USER" "$BACKUP_DIR"

echo "=== Starte Backup-Prozess: $(date) ==="

# 1. Alle laufenden Docker-Projekte in den Unterordnern stoppen
echo "Suche und stoppe alle Docker-Container..."
find "$SOURCE_DIR" -name "docker-compose.yml" | while read -r compose_file; do
    echo "Stoppe Projekt in: $(dirname "$compose_file")"
    cd "$(dirname "$compose_file")" && docker compose down
done

# 2. Backup erstellen
echo "Erstelle komprimiertes Archiv von $SOURCE_DIR..."
tar -czf "$BACKUP_DIR/$BACKUP_NAME" -C "$SOURCE_DIR" .

# 3. Alle Docker-Projekte wieder hochfahren
echo "Suche und starte alle Docker-Container neu..."
find "$SOURCE_DIR" -name "docker-compose.yml" | while read -r compose_file; do
    echo "Starte Projekt in: $(dirname "$compose_file")"
    cd "$(dirname "$compose_file")" && docker compose up -d
done

# 4. Alte Backups löschen (älter als 7 Tage)
echo "Räume alte Backups auf (älter als 7 Tage)..."
find "$BACKUP_DIR" -type f -name "docker_backup_*.tar.gz" -mtime +7 -delete

# Rechte für den Backup-Ordner korrigieren
chown -R "$SUDO_USER:$SUDO_USER" "$BACKUP_DIR"

echo "=== Backup erfolgreich abgeschlossen: $(date) ==="

```bash

### 3. Automatisierung einrichten
Damit wir nicht das Backup jedesmal selber manuel starten müssen, wird es automatisiert

0 3 * * * /home/ramed/docker/backup.sh > /dev/null 2>&1
