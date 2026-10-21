---
title: WSL auf Windows – Best Practice Setup
type: knowledge
created: 2026-10-21
updated: 2026-10-21
status: draft
confidence: sourced
tags: wsl, windows, setup, mslearn
sources: "[[#Quellen]]"
---

# WSL auf Windows – Best Practice Setup

Wissensbasis für den `wsl-setup`-Agenten. Alle Aussagen stammen aus Microsoft
Learn (abgerufen 2026-10-21). Der Agent muss sie vor Anwendung per
`microsoft_docs_fetch` gegen die Live-Doku prüfen, da WSL-Optionen sich schnell
ändern.

> [!warning] Systemabhängigkeit
> Kein Schritt gilt pauschal. Erst das System des Nutzers erfassen
> ([[#1. Systemprofil erfassen]]), dann die passenden Zweige wählen.

## 1. Systemprofil erfassen

Nur lesende Befehle, vor jeder Änderung ausführen und Ergebnis dem Nutzer zeigen.

| Frage | PowerShell | Warum |
| --- | --- | --- |
| Windows-Version/Build | `winver` bzw. `[Environment]::OSVersion` | Win10 ≥ Build 19041 oder Win11 nötig; mirrored/dnsTunneling/autoProxy nur Win11 22H2+ |
| RAM, CPU-Kerne | `Get-CimInstance Win32_ComputerSystem` | Grundlage für `memory`/`processors` |
| Virtualisierung aktiv | `systeminfo` (Hyper-V-Anforderungen) | WSL 2 braucht Virtualisierung in BIOS/UEFI |
| WSL-Stand | `wsl --version`, `wsl -l -v` | `--version` fehlt → veraltete Inbox-Version, `wsl --update` |
| VPN/Proxy/Firewall | Nutzer fragen | Entscheidet NAT vs. mirrored, DNS-Tunneling, autoProxy |
| Zweck (Web, Docker, ML/GPU, C++) | Nutzer fragen | Entscheidet Zusatzmodule |
| Unternehmens-PC (Intune/Defender)? | Nutzer fragen | Dann [[#8. Unternehmensumfeld]] |

## 2. Installation

1. PowerShell als Administrator: `wsl --install` (aktiviert WSL + Virtual Machine
   Platform, installiert Kernel, setzt WSL 2 als Standard, installiert Ubuntu).
2. Neustart.
3. Andere Distribution: `wsl --list --online`, dann `wsl --install -d <Distro>`.
   Hängt die Installation bei 0 %: `wsl --install --web-download -d <Distro>`.
4. Ersten Start abwarten, Linux-Benutzer und Passwort anlegen (unabhängig vom
   Windows-Konto, Eingabe erscheint nicht auf dem Bildschirm).
5. Prüfen: `wsl -l -v` → VERSION muss `2` sein; sonst
   `wsl --set-version <Distro> 2`.

## 3. Nach der Installation

- Pakete selbst aktuell halten (Windows tut das nicht):
  `sudo apt update && sudo apt upgrade`.
- WSL selbst aktuell halten: `wsl --update`.
- Windows Terminal als Standardkonsole verwenden (Tabs, Panes, Profile pro Distro).
- Passwort vergessen: `wsl -d <Distro> -u root`, dann `passwd <user>`.

## 4. Dateispeicherung (größter Performance-Hebel)

- Projekte für Linux-Tools gehören ins Linux-Dateisystem
  (`~/projects`, von Windows als `\\wsl$\<Distro>\home\<user>\projects`).
- **Nicht** unter `/mnt/c/...` arbeiten: 9P-Übersetzung bremst stark.
- Umgekehrt: Windows-Tools → Dateien auf NTFS.
- `explorer.exe .` öffnet das aktuelle Verzeichnis in Windows.
- Pfade nie hartkodieren, `wslpath` nutzen (`/mnt/`-Root ist über
  `[automount] root` änderbar).
- `\\wsl$\` existiert nur, solange die Distro läuft.

## 5. Konfiguration

Zwei Ebenen: `%UserProfile%\.wslconfig` (global, WSL 2) und `/etc/wsl.conf`
(pro Distro). Änderungen greifen erst nach `wsl --shutdown` und ca. 8 Sekunden
Wartezeit. Alternativ „WSL Settings“ im Startmenü.

### 5.1 Empfohlene `.wslconfig` (Werte ans Systemprofil anpassen)

```ini
[wsl2]
# Faustregel: 50 % des RAM ist Standard; bei <16 GB RAM bewusst begrenzen
memory=8GB
processors=4
# Win11 22H2+: bessere VPN-/Firewall-Kompatibilität
networkingMode=mirrored
dnsTunneling=true
autoProxy=true
firewall=true

[experimental]
autoMemoryReclaim=gradual
sparseVhd=true
```

Regeln:

- `memory`/`processors` nur setzen, wenn Standardwerte stören; nie über
  Windows-Bedarf hinaus.
- `networkingMode=mirrored`, `dnsTunneling`, `autoProxy` nur auf Windows 11
  22H2+. Auf Windows 10 weglassen (NAT bleibt).
- Pfade in `.wslconfig` mit doppeltem Backslash: `C:\\Temp\\x`.
- Datei fehlerhaft → WSL startet trotzdem, aber ohne Einstellungen.

### 5.2 Empfohlene `/etc/wsl.conf`

```ini
[boot]
systemd=true

[interop]
appendWindowsPath=false   # optional: schneller, sauberer $PATH

[automount]
options="metadata"        # nur wenn Linux-Rechte auf Windows-Dateien nötig
```

- `systemd=true` setzt WSL ≥ Store-Version voraus (`wsl --version`).
- `appendWindowsPath=false` verhindert Windows-Binaries im PATH; dann
  `.exe`-Tools mit vollem Pfad aufrufen. Bewusst entscheiden.

## 6. Netzwerk: Entscheidungsbaum

| Situation | Empfehlung |
| --- | --- |
| Windows 10 | NAT (Standard); Windows-Host-IP per `ip route show \| grep default` |
| Windows 11 22H2+, normales Netz | `mirrored` + `dnsTunneling=true` (localhost funktioniert in beide Richtungen) |
| VPN im Einsatz | `dnsTunneling=true` (VPN-NRPT-Regeln greifen nur damit) |
| Firmen-Proxy | `autoProxy=true` |
| `.local`-Namen (mDNS) | mirrored + `libnss-mdns` + `mdns_minimal` in `/etc/nsswitch.conf` (WSL ≥ 2.3.17) |
| Docker Desktop mit published ports + mirrored | `--network host` oder Port in `ignoredPorts` |
| Bekannt inkompatible VPNs in mirrored | Bitdefender, OpenVPN 2.6.501, McAfee Safe Connect → NAT verwenden |
| Cisco AnyConnect + NAT | Workaround aus der Cisco-Doku nötig |
| Global Secure Access Client | `dnsTunneling=false` oder `networkingMode=nat` |

## 7. Entwicklungs-Module (nach Zweck zuschalten)

- **VS Code**: Erweiterung „WSL“, Start mit `code .` aus dem Linux-Projektordner.
- **Git**: Git in WSL installieren; Windows Credential Manager als
  Credential-Helper; Zeilenenden (`core.autocrlf`) bewusst setzen.
- **Docker**: Docker Desktop mit WSL-2-Backend und Distro-Integration.
- **Datenbanken**: MySQL, PostgreSQL, MongoDB, Redis, SQL Server, SQLite
  laufen direkt in der Distro (mit `systemd=true` als Dienst).
- **GPU/ML**: Windows-Treiber installieren, keinen Linux-GPU-Treiber in WSL;
  `[gpu] enabled=true` ist Standard.
- **GUI-Apps**: WSLg (`guiApplications=true`, Standard).
- **Externe Laufwerke**: `wsl --mount`.

## 8. Unternehmensumfeld

Ab Windows 11 22H2 und WSL 2.0.9 gelten Windows-Firewall-Regeln automatisch
für WSL. Empfohlen: `mirrored`, `dnsTunneling=true`, `autoProxy=true`,
Hyper-V-Firewall konfigurieren, Verwaltung über Defender for Endpoint/Intune.

## 9. Wartung und Diagnose

| Aktion | Befehl |
| --- | --- |
| Alles neu starten | `wsl --shutdown` |
| Einzelne Distro stoppen | `wsl --terminate <Distro>` |
| Laufende Distros | `wsl --list --running` |
| Update | `wsl --update` (Vorab: `--pre-release`) |
| Backup | `wsl --export <Distro> <datei.tar>` |
| Restore | `wsl --import <Name> <Pfad> <datei.tar>` |
| Wiederherstellung | `safeMode=true` in `.wslconfig` (Win11) |

> [!danger] Destruktiv
> `wsl --unregister <Distro>` löscht die gesamte Distro samt Daten. Nur nach
> `wsl --export` und ausdrücklicher Bestätigung des Nutzers.

## 10. Häufige Fehler (laut Microsoft Learn)

1. Projekte unter `/mnt/c` für Linux-Tools ablegen.
2. `/mnt/c` hartkodieren statt `wslpath`.
3. `.exe` beim Aufruf von Windows-Programmen aus WSL vergessen.
4. Netzwerk-Topologie annehmen; NAT und mirrored verhalten sich verschieden.
5. WSL 1-Annahmen auf WSL 2 übertragen.
6. `wsl.conf`-Änderung ohne `wsl --shutdown` testen.
7. Nicht prüfen, ob `wsl.exe` überhaupt existiert (Server, Firmen-PCs).

## Quellen

- https://learn.microsoft.com/windows/wsl/install
- https://learn.microsoft.com/windows/wsl/setup/environment
- https://learn.microsoft.com/windows/wsl/wsl-config
- https://learn.microsoft.com/windows/wsl/networking
- https://learn.microsoft.com/windows/wsl/troubleshooting
- https://learn.microsoft.com/windows/wsl/enterprise
- https://learn.microsoft.com/windows/dev-environment/wsl-interop
