# Vaultwarden auf dem Raspberry Pi

Self-hosted Vaultwarden auf einem Raspberry Pi 4 – Schritt für Schritt aufgebaut als Lern- und Portfolio-Projekt.

Ziel dieses Projekts ist nicht nur, einen eigenen Passwortmanager zu betreiben, sondern dabei die zugrunde liegenden Themen rund um Linux, Docker, Netzwerke, Persistenz, HTTPS und Sicherheit wirklich zu verstehen.

## Aktuelle Architektur

```text
Mac / iPhone / iPad / Windows
             │
          HTTPS
             │
      Tailscale Serve
             │
      127.0.0.1:8000
             │
      Vaultwarden
      Docker-Container
             │
   /srv/vaultwarden/data
```

Vaultwarden wird nicht direkt im Heimnetz veröffentlicht. Der Container-Port ist ausschließlich an das Loopback-Interface des Raspberry Pi gebunden.

## Aktueller Stand

- Debian 13 (trixie), ARM64
- Docker Engine + Docker Compose aus dem offiziellen Docker-Repository
- Vaultwarden `1.37.3`
- Container läuft mit einer dedizierten Non-Root-UID/GID (`102:105` auf diesem Host)
- Persistente Daten liegen außerhalb des Git-Repositories unter `/srv/vaultwarden/data`
- Datenverzeichnis ist auf den dedizierten Servicebenutzer `vaultwarden` beschränkt
- Vaultwarden ist nur an `127.0.0.1:8000` gebunden
- HTTPS-Zugriff über Tailscale
- Neue Registrierungen wurden nach Erstellung des ersten Accounts deaktiviert
- Container-Healthcheck erfolgreich als `healthy` geprüft
- Persistenz durch Löschen und Neuerstellen des Containers getestet

## Repository-Struktur

```text
.
├── compose.yaml
├── .gitignore
└── README.md
```

Laufzeitdaten, Backups und Secrets werden bewusst außerhalb von Git gehalten.

## Docker Compose

Die aktuelle Compose-Konfiguration verwendet:

- eine fest gepinnte Vaultwarden-Version statt `latest`
- `restart: unless-stopped`
- eine dedizierte Non-Root-UID/GID
- ausschließlich lokales Port-Binding über Loopback
- einen Bind Mount für persistente Daten
- deaktivierte öffentliche Registrierungen

Das Datenverzeichnis auf dem Host lautet:

```text
/srv/vaultwarden/data
```

Im Container ist dieses Verzeichnis eingebunden als:

```text
/data
```

## Sicherheitsentscheidungen

Dieses Projekt trennt Konfiguration bewusst von sensiblen Laufzeitdaten.

Folgende Inhalte dürfen **niemals** in dieses Repository committed werden:

```text
.env
Vaultwarden-Datenbankdateien
RSA-/Private-Keys
Backups
API-Tokens
sonstige Secrets
```

Die `.gitignore` schließt bereits typische Secret- und Laufzeitpfade aus.

Das Vaultwarden-Datenverzeichnis auf dem Host ist ausschließlich für den Servicebenutzer zugänglich.

> Hinweis: Die UID/GID in `compose.yaml` ist host-spezifisch. Bei einer anderen Installation sollte ein eigener Servicebenutzer angelegt und dessen numerische UID/GID verwendet werden.

## Nützliche Prüfungen

Compose-Konfiguration validieren:

```bash
docker compose config
```

Laufenden Dienst prüfen:

```bash
sudo docker compose ps
```

Letzte Vaultwarden-Logs anzeigen:

```bash
sudo docker compose logs --tail=30 vaultwarden
```

Lokalen HTTP-Endpunkt auf dem Raspberry Pi testen:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8000/
```

Eine erfolgreiche Antwort liefert aktuell HTTP `200`.

## Geplante nächste Schritte

- Tailscale-/HTTPS-Setup ausführlicher dokumentieren
- Backup-Strategie erstellen und testen
- Update-Prozess definieren
- weiteres Security-Hardening
- Restore-Prozess dokumentieren
- Bitwarden-Clients unter macOS, iOS, iPadOS und Windows testen
- Repository vor einer späteren Veröffentlichung nochmals gezielt auf Secrets prüfen

## Projektziel

Dieses Repository ist in erster Linie ein Lern- und Portfolio-Projekt. Das Setup wird bewusst manuell und schrittweise aufgebaut, damit jede Komponente und jede Sicherheitsentscheidung nachvollzogen und verstanden wird, statt nur einen fertigen Stack zu kopieren.

Vaultwarden ist eine inoffizielle, Bitwarden-kompatible Serverimplementierung und nicht mit Bitwarden, Inc. verbunden.
