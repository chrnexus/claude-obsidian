---
title: Plan – Hostinger Linux-VPS für Web und Docker, WSL später
type: knowledge
created: 2026-10-21
updated: 2026-10-21
status: draft
confidence: mixed
tags: hostinger, vps, linux, docker, web, wsl
sources: "[[#Quellen]]"
---

# Plan: Hostinger Linux-VPS, WSL erst mit Win-11-PC

Ersetzt den früheren Windows-11-Plan. Grundlage: Es gibt noch **keinen
Windows-11-PC**. Gehostet wird auf einem **Linux-VPS bei Hostinger**. WSL ist
damit nicht die Zielumgebung, sondern später die lokale Entwicklungsumgebung,
die der VPS-Distribution gleicht ([[wsl-windows-best-practice]]).

> [!warning] Quellenlage
> Tarife, Preise, Rechenzentren und Vorlagen stammen aus der Hostinger-API
> (Connector, nur lesend) vom 2026-10-21. Textpassagen zu Web/Cloud-Hosting
> stammen aus Websuchergebnissen.
> Preise und Specs vor dem Kauf auf der Live-Seite prüfen. Abschnitte mit
> `[allgemein]` sind Linux-Standardpraxis und nicht aus Microsoft Learn oder
> Hostinger belegt.

## 1. Hostinger-Produkte im Überblick

| Produkt | Zweck | Eignung hier |
| --- | --- | --- |
| Web Hosting (Business, ab ca. $3.99/Monat, Promo) | Verwaltetes Hosting, bis 5 Node.js-Apps | Kein Docker, kein Root |
| Cloud Hosting (Cloud Startup, ca. $7.99/Monat Promo) | Verwaltet, 4 GB RAM, 10 Node.js-Apps | Kein Docker, kein Root |
| Node.js-Hosting (verwaltet) | Node-Apps ohne Serverpflege | Nur für reine Node-Apps |
| **KVM-VPS (KVM 1/2/4/8)** | Root-Zugriff, eigenes Linux, Docker möglich | **Passt zur Anforderung** |

Die Anforderung „Linux-Server, Docker, eigene Kontrolle“ führt zum KVM-VPS.
Preise und Konditionen schwanken je nach Seite, Währung und Laufzeit;
Promopreise steigen bei Verlängerung.

### KVM-Tarife (Hostinger-API, Katalog `hostingerde`, Abruf 2026-10-21)

Preise in EUR; links erste Laufzeit, rechts regulärer Preis (Verlängerung).
Alle Tarife: 300 MB/s Netzwerk.

| Tarif | vCPU | RAM | NVMe | Traffic | monatlich | 1 Jahr | 2 Jahre |
| --- | --- | --- | --- | --- | --- | --- | --- |
| KVM 1 | 1 | 4 GB | 50 GB | 4 TB | 7,99 → 17,49 | 71,88 → 155,88 | 131,76 → 287,76 |
| KVM 2 | 2 | 8 GB | 100 GB | 8 TB | 9,99 → 21,49 | 101,88 → 203,88 | 186,96 → 359,76 |
| KVM 4 | 4 | 16 GB | 200 GB | 16 TB | 19,99 → 35,99 | 155,88 → 359,88 | 263,76 → 671,76 |
| KVM 8 | 8 | 32 GB | 400 GB | 32 TB | 38,99 → 64,99 | 299,88 → 647,88 | 527,76 → 1199,76 |

Die Game-Panel-Tarife (Minecraft) sind für dein Vorhaben irrelevant.
Der Verlängerungspreis liegt deutlich über dem Einstiegspreis; Laufzeit
bewusst wählen.

Rechenzentren: Vilnius, Paris (2), Manchester, Düsseldorf in Europa; weitere in
Nordamerika, Asien, Südamerika. Für Nutzer in Deutschland liegt Düsseldorf nahe.

Relevante OS-Vorlagen (IDs der API): Ubuntu 24.04 mit Docker (1121), mit
Docker und Traefik (1210), Ubuntu 24.04 (1077), Debian 12/13, Dokploy (1092),
Coolify (1087), Portainer (1143), Easypanel (1122), n8n (1153), Supabase (1199),
MERN/MEAN/MEVN (1206/1205/1207), Ubuntu 24.04 mit Claude Code (1189).

### Empfehlung (Einschätzung, nicht Quellenaussage)

**KVM 2** als Start: 8 GB RAM tragen Docker, Datenbank und Build-Prozesse.
KVM 1 (4 GB) wird mit Docker plus Node-Builds schnell eng. Wegen Snapshots
und Tarifwechsel lässt sich später hochskalieren.

## 2. Reihenfolge

1. Tarif und Laufzeit wählen (Live-Preis prüfen).
2. Beim Einrichten die Vorlage **Ubuntu 24.04 mit Docker** wählen.
3. SSH-Schlüssel hinterlegen, Passwort-Login später abschalten.
4. Firewall in der Hostinger-Verwaltung und auf dem Server konfigurieren.
5. Härtung und Updates ([[#3. Server-Grundsetup]]).
6. Anwendungen per Docker Compose, Traefik für HTTPS.
7. Backups und Snapshot-Routine ([[#5. Betrieb]]).
8. Später: Win-11-PC mit WSL, gleiche Ubuntu-Version, gleiche Compose-Dateien.

## 3. Server-Grundsetup [allgemein]

```bash
# als root, einmalig
apt update && apt upgrade -y
adduser deploy && usermod -aG sudo,docker deploy
# SSH-Schlüssel für deploy hinterlegen, dann in /etc/ssh/sshd_config:
#   PermitRootLogin no
#   PasswordAuthentication no
systemctl reload ssh
ufw allow OpenSSH && ufw allow 80,443/tcp && ufw enable
apt install -y unattended-upgrades fail2ban
```

Vor dem Abschalten von Root- und Passwort-Login in einer **zweiten** Sitzung
prüfen, dass der Schlüssel-Login funktioniert, sonst sperrst du dich aus.
Docker-Hinweis: Docker umgeht UFW-Regeln bei veröffentlichten Ports. Nur 80/443
über Traefik veröffentlichen, Datenbanken an kein öffentliches Interface binden.

## 4. Deployment [allgemein]

- Compose-Projekt pro Anwendung unter `/srv/<app>`, Secrets in `.env`
  (nie ins Git).
- Traefik als einziger Eintrittspunkt für 80/443 mit automatischem TLS.
- Images mit festem Tag, nicht `latest`.
- Deployment aus Git per Pull auf dem Server oder CI mit SSH-Schlüssel.

## 5. Betrieb

- Hostinger-Snapshot vor jeder größeren Änderung; wöchentliche Backups
  ersetzen keine Datenbank-Dumps.
- Docker-Aufräumen: `docker system prune` nur bewusst, nie mit `--volumes`
  ohne Backup.
- Monitoring: `docker stats`, `df -h`, Logs rotieren.
- Domain: A/AAAA-Record auf die VPS-IP.

## 6. Lokale Entwicklung, sobald der Win-11-PC da ist

Dann gilt [[wsl-windows-best-practice]] mit diesen Anpassungen:

- Distribution: **Ubuntu 24.04**, identisch zum VPS.
- Projekte im Linux-Dateisystem (`~/projects`), nicht `/mnt/c`.
- Deployment testen mit denselben Compose-Dateien; Docker Desktop oder
  `wslc.exe` (laut Learn ab WSL 2.9.3).
- Bei VPN: `dnsTunneling=true`, mirrored nur mit kompatiblem VPN.
- 16 GB RAM waren als Annahme genannt: `memory=8GB` in `.wslconfig`.

## Quellen

- https://www.hostinger.com/vps-hosting
- https://www.hostinger.com/support/6976044-parameters-and-limits-of-hosting-plans-in-hostinger/
- https://www.hostinger.com/docker-hosting
- https://www.hostinger.com/ubuntu-hosting (Ubuntu-VPS-Seite)
- https://www.hostinger.com/nodejs-hosting
- https://www.hostinger.com/pricing
- https://learn.microsoft.com/windows/wsl/ (für Abschnitt 6)
