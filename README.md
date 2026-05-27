Markdown
# MyLab – Private Cloud & Security Infrastructure

Dieses Repository dokumentiert den Aufbau und die Konfiguration meiner privaten Server-Infrastruktur auf Basis von Ubuntu Server, Docker und Docker Compose.

## Architektur-Übersicht
Die Infrastruktur ist so konzipiert, dass alle Dienste isoliert in Docker-Containern laufen und über einen zentralen Einstiegspunkt abgesichert sind.

*   **Reverse Proxy:** Nginx Proxy Manager (Sorgt für das Routing und blockiert unbefugte Anfragen)
*   **SSL-Verschlüsselung:** Automatisierte Zertifikatsverwaltung via Let's Encrypt (Force SSL / HTTP/2)
*   **Passwort-Management:** Vaultwarden (Bitwarden-API in Rust) zur sicheren, selbstgehosteten Passwortverwaltung
*   **Cloud-Speicher:** Nextcloud (Zentrale Datensynchronisation) gekoppelt an eine dedizierte MariaDB-Datenbank

---

## Docker Compose Konfigurationen

### 1. Reverse Proxy & Vaultwarden
Hierüber wird der Nginx Proxy Manager als zentraler Pförtner sowie die Vaultwarden-Instanz betrieben.

```yamlversion: '3.8'

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

