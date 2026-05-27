Markdown
# MyLab – Private Cloud & Security Infrastructure

Dieses Repository dokumentiert den Aufbau, die Netzwerkstruktur und die Konfiguration meiner privaten Server-Infrastruktur auf Basis von Ubuntu Server, Docker und Docker Compose.

## Architektur-Übersicht
Die Infrastruktur ist so konzipiert, dass alle Dienste isoliert in Docker-Containern laufen und über einen zentralen Einstiegspunkt abgesichert sowie geroutet werden.

*   **Reverse Proxy:** Nginx Proxy Manager (Zentraler Pförtner, der den externen Traffic auf Port 80/443 kontrolliert und an die internen Container-Ports weiterleitet).
*   **SSL-Verschlüsselung:** Automatisierte, dynamische Zertifikatsverwaltung via Let's Encrypt mit erzwungenem HTTPS (Force SSL) und HTTP/2-Unterstützung.
*   **Passwort-Management:** Vaultwarden (Bitwarden-API in Rust) zur hochsicheren, ressourcenschonenden Verwaltung von Anmeldedaten.
*   **Cloud-Speicher:** Nextcloud (Zentrale Datensynchronisation) gekoppelt an eine dedizierte MariaDB-Datenbank für maximale Performance bei Dateioperationen.

---

## Docker-Netzwerk & Kommunikation
Damit der Reverse Proxy den Datenverkehr an die Anwendungen weiterleiten kann, ohne dass alle Container ihre Ports direkt an das Host-System freigeben müssen, nutzen die Container dedizierte Docker-Netzwerke (`bridge`). 

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
2. Passwort-Tresor (Vaultwarden)
Sichere Passwort-Instanz, die intern auf Port 8080 lauscht und vom Proxy per SSL geschützt wird.

YAML
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
3. Cloud-Zentrale & Relationale Datenbank (Nextcloud & MariaDB)
Die Nextcloud-Anwendung läuft isoliert und kommuniziert intern direkt mit dem MariaDB-Datenbank-Container.

YAML
version: '3.8'

services:
  nextcloud-db:
    image: mariadb:10.11
    container_name: nextcloud-db
    restart: always
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW
    volumes:
      - db_data:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=DATABASE_ROOT_PASSWORD_PLACEHOLDER
      - MYSQL_PASSWORD=DATABASE_USER_PASSWORD_PLACEHOLDER
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud

  nextcloud-app:
    image: nextcloud:latest
    container_name: nextcloud-app
    restart: always
    ports:
      - "8090:80"
    depends_on:
      - nextcloud-db
    volumes:
      - nextcloud_data:/var/www/html
    environment:
      - MYSQL_PASSWORD=DATABASE_USER_PASSWORD_PLACEHOLDER
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=nextcloud-db

volumes:
  db_data:
  nextcloud_data:
Security Hardening & Infrastruktur-Schutz
Daten-Persistenz: Kritische Daten (Anwendungsdaten und Datenbanken) werden über benannte Docker-Volumes (volumes) und lokale Verzeichnisse auf dem Host-System gesichert, um Datenverlust bei Container-Updates zu verhindern.

Access Control: Unmittelbar nach dem Deployment wurden sämtliche Standard-Admin-Credentials der Weboberflächen modifiziert und gehärtet.

Port-Absicherung: Nur die Ports 80 (HTTP) und 443 (HTTPS) sowie der administrative NPM-Port 81 sind nach außen hin geöffnet. Die Anwendungen selbst (Vaultwarden, Nextcloud) kommunizieren geschützt hinter dem Proxy.

Troubleshooting & Learnings (DevOps Skills)
ACME-Validierung bei Let's Encrypt: Erste Zertifikatsanfragen schlugen fehl, da der ACME-Server Standard-Dummy-E-Mail-Adressen (admin@example.com) blockiert. Die Fehleranalyse erfolgte direkt über die Docker-Container-Logs (docker logs). Das Problem wurde durch die Konfiguration einer validen Mail-Struktur im globalen Administrator-Profil behoben.

DNS-Routing: Konfiguration von spezifischen A-Records (passworte.*, cloud.*) im Domain-Management, um Subdomains gezielt auf die Server-IP aufzulösen, ohne bestehende Produktivsysteme auf der Hauptdomain zu beeinträchtigen.



