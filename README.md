# Vaultwarden on Raspberry Pi

Self-hosted Vaultwarden setup on a Raspberry Pi 4, built step by step as a learning and portfolio project.

The goal of this project is not only to run a password manager, but to understand the underlying Linux, Docker, networking, persistence, HTTPS and security concepts.

## Current architecture

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
      Docker container
             │
   /srv/vaultwarden/data
```

Vaultwarden itself is not exposed directly to the local network. The container port is bound only to the loopback interface of the Raspberry Pi.

## Current status

- Debian 13 (trixie), ARM64
- Docker Engine + Docker Compose installed from the official Docker repository
- Vaultwarden `1.37.3`
- Container runs as a dedicated non-root UID/GID (`102:105` on the current host)
- Persistent data stored outside the repository in `/srv/vaultwarden/data`
- Data directory restricted to the dedicated `vaultwarden` service account
- Vaultwarden bound to `127.0.0.1:8000`
- HTTPS access through Tailscale
- New account registrations disabled after the initial account was created
- Container health check verified as `healthy`
- Persistence tested by deleting and recreating the container

## Repository structure

```text
.
├── compose.yaml
├── .gitignore
└── README.md
```

Runtime data, backups and secrets are deliberately kept outside Git.

## Docker Compose

The current Compose configuration uses:

- a pinned Vaultwarden image version instead of `latest`
- `restart: unless-stopped`
- a dedicated non-root UID/GID
- loopback-only port publishing
- a bind mount for persistent data
- disabled public sign-ups

The host-side data directory is:

```text
/srv/vaultwarden/data
```

The container sees this directory as:

```text
/data
```

## Security decisions

This project intentionally separates configuration from sensitive runtime data.

The following must **never** be committed to this repository:

```text
.env
Vaultwarden database files
RSA/private keys
backups
API tokens
other secrets
```

The `.gitignore` already excludes common secret and runtime paths.

The Vaultwarden data directory on the host is restricted with owner-only permissions.

> Note: the UID/GID in `compose.yaml` is host-specific. A different installation should create its own dedicated service account and use that account's numeric UID/GID.

## Useful checks

Validate the Compose configuration:

```bash
docker compose config
```

Check the running service:

```bash
sudo docker compose ps
```

View recent Vaultwarden logs:

```bash
sudo docker compose logs --tail=30 vaultwarden
```

Test the local HTTP endpoint on the Raspberry Pi:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8000/
```

A successful response currently returns HTTP `200`.

## Planned next steps

- document the Tailscale HTTPS setup in more detail
- create and test a backup strategy
- define an update procedure
- continue security hardening
- document restore procedures
- test Bitwarden clients on macOS, iOS, iPadOS and Windows
- review the repository for secrets before making it public

## Project purpose

This repository is primarily a learning and portfolio project. The setup is being built manually and incrementally so that every component and security decision is understood rather than copied as a finished stack.

Vaultwarden is an unofficial Bitwarden-compatible server implementation and is not affiliated with Bitwarden, Inc.
