# Vaultwarden auf dem Raspberry Pi

Self-hosted Vaultwarden auf einem Raspberry Pi 4 – Schritt für Schritt aufgebaut als Lern- und Portfolio-Projekt.

Ziel dieses Projekts ist nicht nur, einen eigenen Passwortmanager zu betreiben, sondern die zugrunde liegenden Themen rund um Linux, Docker, Netzwerke, VPN, DNS, HTTPS, Reverse Proxies, Persistenz und Sicherheit praktisch zu verstehen.

## Architektur

```text
MacBook / iPhone / iPad
          │
          │ HTTPS
          │
   WireGuard-VPN
          │
          ▼
      FRITZ!Box
          │
          ▼
  Raspberry Pi 4
          │
       Port 443
          │
          ▼
        Caddy
    Reverse Proxy
          │
     Docker-Netzwerk
          │
          ▼
      Vaultwarden
        Port 80
          │
          ▼
 /srv/vaultwarden/data
```

WireGuard läuft auf der FRITZ!Box und stellt den sicheren Zugang zum Heimnetz bereit.

Vaultwarden selbst veröffentlicht keinen Port am Host. Nur Caddy ist über HTTPS auf Port `443` erreichbar und leitet Anfragen innerhalb des Docker-Netzwerks an Vaultwarden weiter.

Es bestehen keine öffentlichen Portfreigaben für Vaultwarden.

## Aktueller Stand

- Raspberry Pi 4 mit Debian 13 (trixie), ARM64
- Docker Engine und Docker Compose
- Vaultwarden `1.37.3`
- Caddy als Reverse Proxy
- HTTPS über eine interne Caddy-CA
- WireGuard-VPN über die FRITZ!Box
- DNS-basierter Zugriff über eine eigene Subdomain
- Vaultwarden ausschließlich innerhalb des Docker-Netzwerks erreichbar
- Caddy veröffentlicht ausschließlich HTTPS auf Port `443`
- Persistente Vaultwarden-Daten unter `/srv/vaultwarden/data`
- Container läuft mit einer dedizierten Non-Root-UID/GID
- Neue Registrierungen deaktiviert
- Zwei-Faktor-Authentifizierung aktiviert
- Zugriff über macOS, iOS und iPadOS getestet
- Synchronisation zwischen mehreren Clients erfolgreich getestet
- Zugriff über Mobilfunk und WireGuard erfolgreich getestet

## Repository-Struktur

```text
.
├── .env.example
├── .gitignore
├── Caddyfile
├── compose.yaml
└── README.md
```

Produktive Konfiguration, Zertifikate, Vault-Daten und andere sensible Dateien werden bewusst nicht im Repository gespeichert.

## Konfiguration

Die produktive Domain wird über eine lokale `.env`-Datei bereitgestellt.

Beispiel:

```env
VAULT_DOMAIN=vault.example.com
```

Die Datei `.env` wird von Git ignoriert.

Als Vorlage befindet sich `.env.example` im Repository.

Dadurch bleibt die produktive Konfiguration vom veröffentlichten Projekt getrennt.

## Docker Compose

Die Compose-Konfiguration verwendet:

- eine fest gepinnte Vaultwarden-Version statt `latest`
- `restart: unless-stopped`
- eine dedizierte Non-Root-UID/GID
- einen Bind Mount für persistente Vaultwarden-Daten
- Caddy als separaten Reverse-Proxy-Container
- interne Docker-Kommunikation zwischen Caddy und Vaultwarden
- keine Veröffentlichung des Vaultwarden-Ports auf dem Host
- deaktivierte öffentliche Registrierungen

Das persistente Datenverzeichnis auf dem Host lautet:

```text
/srv/vaultwarden/data
```

Im Vaultwarden-Container wird es eingebunden als:

```text
/data
```

## HTTPS und Caddy

Caddy übernimmt die TLS-Terminierung und leitet HTTPS-Anfragen intern an Vaultwarden weiter.

Das Caddyfile verwendet die Domain aus einer Umgebungsvariable:

```caddyfile
https://{$VAULT_DOMAIN} {
    tls internal
    reverse_proxy vaultwarden:80
}
```

Mit `tls internal` betreibt Caddy eine eigene lokale Certificate Authority.

Das Root-Zertifikat dieser CA muss auf den verwendeten Clients als vertrauenswürdig installiert werden.

Root-Zertifikate und insbesondere private CA-Schlüssel gehören nicht in dieses Repository.

## WireGuard

Der Raspberry Pi betreibt keinen eigenen WireGuard- oder Tailscale-Dienst.

WireGuard wird direkt von der FRITZ!Box bereitgestellt.

MacBook, iPhone und iPad verbinden sich zunächst per WireGuard mit dem Heimnetz und greifen anschließend über HTTPS auf Vaultwarden zu.

Dadurch muss Vaultwarden nicht direkt aus dem öffentlichen Internet erreichbar sein.

## DNS

Die verwendete Vaultwarden-Subdomain wird per DNS auf die private IPv4-Adresse des Raspberry Pi aufgelöst.

Da eine öffentliche Domain auf eine private Adresse zeigt, kann bei einer FRITZ!Box eine gezielte DNS-Rebind-Ausnahme für die Vaultwarden-Domain erforderlich sein.

Die produktive Domain wird nicht im Repository gespeichert.

## Sicherheitsentscheidungen

Folgende Inhalte dürfen niemals in dieses Repository committed werden:

```text
.env
*.crt
*.key
*.pem
*.p12
*.pfx
WireGuard-Konfigurationen
Vaultwarden-Datenbankdateien
Backups
API-Tokens
Secrets
private CA-Schlüssel
```

Die `.gitignore` schließt diese Dateitypen und Verzeichnisse aus.

Vaultwarden selbst besitzt kein öffentliches Port-Mapping.

Der frühere Test-Port `8000` wurde nach Einrichtung des Reverse Proxys vollständig entfernt.

Von außen erfolgt der Zugriff ausschließlich über:

```text
WireGuard → FRITZ!Box → HTTPS/Caddy → Vaultwarden
```

## Nützliche Prüfungen

Compose-Konfiguration validieren:

```bash
sudo docker compose config
```

Containerstatus prüfen:

```bash
sudo docker compose ps
```

Vaultwarden-Logs anzeigen:

```bash
sudo docker compose logs --tail=30 vaultwarden
```

Caddy-Logs anzeigen:

```bash
sudo docker compose logs --tail=30 caddy
```

HTTPS-Endpunkt testen:

```bash
curl -I https://vault.example.com
```

Eine erfolgreiche Antwort liefert:

```text
HTTP/2 200
```

Prüfen, ob der frühere Port `8000` geschlossen ist:

```bash
sudo ss -tulpn | grep ':8000' || echo "Port 8000 ist geschlossen"
```

## Noch offene Infrastrukturthemen

Die eigentliche Vaultwarden-Installation ist abgeschlossen.

Als getrennte Infrastrukturthemen folgen später:

- Backup-Strategie
- Restore-Test
- Update-Strategie
- Monitoring
- Server- und NAS-Integration

## Projektziel

Dieses Repository ist ein Lern- und Portfolio-Projekt.

Das Setup wurde bewusst manuell und schrittweise aufgebaut, um die einzelnen Komponenten und Sicherheitsentscheidungen zu verstehen, statt lediglich einen fertigen Stack zu kopieren.

Dabei wurden unter anderem praktische Erfahrungen mit Linux, Docker Compose, Docker-Netzwerken, Reverse Proxies, HTTPS/TLS, privaten Certificate Authorities, DNS, WireGuard und sicherer Konfigurationsverwaltung gesammelt.

Vaultwarden ist eine inoffizielle, Bitwarden-kompatible Serverimplementierung und nicht mit Bitwarden, Inc. verbunden.
