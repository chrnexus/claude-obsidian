---
title: WSL-Plan – Windows 11, 16 GB, VPN, Docker, Web-Entwicklung
type: knowledge
created: 2026-10-21
updated: 2026-10-21
status: draft
confidence: sourced
tags: wsl, windows-11, docker, vpn, web
sources: "[[wsl-windows-best-practice#Quellen]]"
---

# WSL-Plan: Windows 11 · 16 GB · VPN · Docker · Web

Zuschnitt von [[wsl-windows-best-practice]] auf das Nutzerprofil. Quellen:
Microsoft Learn (2026-10-21), zusätzlich
https://learn.microsoft.com/windows/wsl/tutorials/wsl-containers und
https://learn.microsoft.com/windows/wsl/tutorials/wsl-git.

> [!warning] Offene Angaben
> Unbekannt und vor Schritt 1 zu klären: Windows-11-Build (mirrored braucht
> 22H2 oder neuer), VPN-Produkt (siehe Inkompatibilitätsliste), CPU-Kerne,
> Docker Desktop vorhanden oder nicht. Werte unten sind Annahmen, wo markiert.

## Schritt 0 – Prüfen (nur lesend, PowerShell)

```powershell
[Environment]::OSVersion.Version          # Build >= 22621 für mirrored/dnsTunneling
wsl --version                              # fehlt der Befehl: wsl --update
wsl -l -v
(Get-CimInstance Win32_ComputerSystem).NumberOfLogicalProcessors
```

## Schritt 1 – Installation

```powershell
wsl --install            # Admin-PowerShell, danach Neustart
wsl --update
```

Ubuntu-Benutzer anlegen, dann `sudo apt update && sudo apt upgrade`.

## Schritt 2 – `%UserProfile%\.wslconfig`

```ini
[wsl2]
# 16 GB RAM: Standard wäre 8 GB (50 %); explizit setzen, Docker + Browser brauchen Luft
memory=8GB
# Annahme: 8 logische Prozessoren -> 6 für WSL. An Schritt 0 anpassen.
processors=6
networkingMode=mirrored     # VPN-Kompatibilität (Win11 22H2+)
dnsTunneling=true           # nötig, damit VPN-NRPT-Regeln in WSL greifen
autoProxy=true              # nur sinnvoll bei Windows-Proxy; sonst weglassen
firewall=true

[experimental]
autoMemoryReclaim=gradual   # Cache zurückgeben, statt RAM zu halten
sparseVhd=true              # VHD schrumpft bei Löschen (Images/node_modules)
```

Danach `wsl --shutdown`, ca. 8 Sekunden warten.

### VPN-Weiche

| Befund | Aktion |
| --- | --- |
| Bitdefender, OpenVPN 2.6.501, McAfee Safe Connect | mirrored ist laut Learn inkompatibel → `networkingMode=nat` |
| Global Secure Access Client | `dnsTunneling=false` oder `networkingMode=nat` |
| Cisco AnyConnect mit NAT | Workaround aus der Cisco-Doku nötig |
| ZScaler | Dauer-Meldung „Http proxy change detected“ bekannt; `autoProxy=false` erwägen |
| Sonst | Konfiguration oben |

Test bei aktivem VPN: `ping`/`curl` auf einen internen und einen externen Host in
WSL. Schlägt DNS fehl, zuerst `dnsTunneling` prüfen.

## Schritt 3 – `/etc/wsl.conf` (in Ubuntu, mit sudo)

```ini
[boot]
systemd=true
```

Optional `appendWindowsPath=false` (sauberer PATH; Windows-Tools dann nur mit
Pfad). Für Web-Entwicklung nicht nötig; Standard belassen, falls `code .`
und `explorer.exe` bequem bleiben sollen.

## Schritt 4 – Projekte und Tools

- Projekte unter `~/projects` im Linux-Dateisystem, nicht `/mnt/c`.
  Das gilt besonders für `node_modules` und Docker-Bind-Mounts.
- Git in WSL installieren, Git for Windows für Git Credential Manager nutzen
  (teilt Anmeldedaten). Proxy-Einstellungen für Git müssen dann in beiden
  `.gitconfig` stehen. GCM funktioniert nur mit HTTPS; für SSH eigene Schlüssel.
- Zeilenenden per `.gitattributes` festlegen, sonst melden Git-Stände
  zwischen Windows/WSL/Container viele Scheinänderungen.
- VS Code mit WSL-Erweiterung; Start per `code .` im Projektordner.
- Windows Terminal als Standard.
- Node per Versionsmanager (nvm/fnm) in WSL, nicht das Windows-Node nutzen.

## Schritt 5 – Docker

- **Docker Desktop**: WSL-2-Backend und Distro-Integration aktivieren.
- Bekannt bei mirrored: Container mit published ports (`-p`) können scheitern.
  Workaround laut Learn: `--network host` oder die Ports in
  `[experimental] ignoredPorts=3000,5173` eintragen (Beispielports anpassen).
- Läuft in der Distro ein Network Manager, Dienst stoppen
  (`sudo systemctl disable network-manager.service`), sonst scheitern
  Loopback-Verbindungen zum Host.
- Container-DNS von Docker Desktop umgeht DNS-Tunneling; Docker hat eigene
  Einstellungen für VPN-DNS.
- **Alternative**: `wslc.exe` ist in neueren WSL-Versionen (laut Learn ab 2.9.3)
  eingebaut, kein Docker Desktop nötig (`wslc run`, `wslc build`).
  Vor Einsatz Version und Kompatibilität mit bestehenden Compose-Workflows
  prüfen; `.wslconfig` gilt nicht für `wslc`-Container.

## Schritt 6 – Verifikation

| Test | Erwartung |
| --- | --- |
| `wsl -l -v` | VERSION 2 |
| `systemctl list-unit-files --type=service` in WSL | systemd läuft |
| `free -h` in WSL | Gesamt ≈ 8 GB |
| `curl -I https://learn.microsoft.com` mit VPN | HTTP 200 |
| `docker run --rm hello-world` | Meldung |
| Dev-Server im WSL auf `localhost:3000`, Aufruf im Windows-Browser | erreichbar |

## Rollback

`.wslconfig` umbenennen oder löschen und `wsl --shutdown`. Distro nie ohne
vorheriges `wsl --export` entfernen.
