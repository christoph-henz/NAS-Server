# Server-Architektur: Ubuntu Server 26.04 LTS


## 1. Physische Hardware

| Komponente | Konfiguration                             |
| ---------- | ----------------------------------------- |
| Mainboard  | ASRock A520M-HDV                          |
| CPU        | AMD Ryzen (AM4)                           |
| RAM        | 32 GB DDR4                                |
| SSD        | Betriebssystem, Anwendungen und Container |
| HDD        | NAS-Daten, Medien und Dateifreigaben      |
| Netzwerk   | 1 Gbit/s Ethernet                         |

## 2. Betriebssystem

Ubuntu Server 26.04 LTS

Ubuntu wird direkt auf der Hardware installiert und verwaltet sämtliche Serverdienste. Es gibt keine zusätzliche Virtualisierungsebene.

## 3. Dienste und Anwendungen

| Dienst               | Aufgabe                         | Umsetzung                                |
| -------------------- | ------------------------------- | ---------------------------------------- |
| Samba                | NAS-Funktion und Dateifreigaben | SMB/CIFS für Windows und Linux           |
| Gameserver           | Spielserver hosten              | Native Dienste oder Docker               |
| Apache / Nginx       | Webserver und APIs              | HTTP/HTTPS, virtuelle Hosts              |
| Datenbanken          | Persistente Anwendungsdaten     | z. B. MariaDB oder PostgreSQL            |
| Homepage             | Homelab-Dashboard               | Zentrale Übersicht und Links             |
| Docker               | Optionale Dienstisolierung      | Container für zusätzliche Anwendungen    |
| Cron / systemd-Timer | Automatisierung                 | Zeitgesteuerte Aufgaben                  |
| Bash-Skripte         | Administration und Backups      | Sicherung, Rotation und Protokollierung  |
| Monitoring           | Systemüberwachung               | CPU, RAM, Speicherplatz und Dienststatus |

## 4. Speicherarchitektur

### SSD – System und Anwendungen

- Ubuntu Server 26.04 LTS
- Anwendungen und Docker-Container
- Konfigurationsdateien und Logs
- Datenbanken und temporäre Daten

### HDD – NAS und Nutzdaten

- Dokumente und persönliche Dateien
- Medien und Spiele
- Samba-Freigaben
- Optional lokale Backupdaten
- ext4 Formatierung (egal ob Windows lesen kann wegen Samba)

Die HDD wird unter Ubuntu unter `/srv/data` eingehängt. Samba stellt ausgewählte Verzeichnisse im Netzwerk bereit.

## 5. Netzwerkarchitektur

### Internes Netzwerk

- Feste IP-Adresse (oder DHCP-Reservierung ; müssen wir schauen) für den Server
- Samba für Windows-Netzlaufwerke
- SSH für die Administration
- Homepage für den zentralen Zugriff auf die Dienste
- Firewall mit gezielten Freigaben

### Externer Zugriff

- Portweiterleitungen nur für tatsächlich benötigte Dienste
- HTTPS für öffentlich erreichbare Webdienste
- VPN, beispielsweise WireGuard, für administrative Zugriffe
- Keine direkte öffentliche Freigabe von SSH oder Samba

## 6. Backupkonzept

| Aufgabe            | Umsetzung                                     |
| ------------------ | --------------------------------------------- |
| Dateisicherung     | `rsync` oder ein versioniertes Backupwerkzeug |
| Datenbanksicherung | Regelmäßige Datenbank-Dumps                   |
| Automatisierung    | Cron oder systemd-Timer                       |
| Aufbewahrung       | Rotation nach definierten Fristen             |
| Protokollierung    | Logdateien und Fehlerstatus                   |
| Benachrichtigung   | E-Mail oder Telegram                          |
| Externe Sicherung  | USB-Festplatte oder separates Netzlaufwerk    |
| Wiederherstellung  | Regelmäßige Restore-Tests                     |

## 7. Zentrale Verwaltung

Homepage dient als Startseite für das Homelab. Dort werden beispielsweise verlinkt:

- Ubuntu-Server und Systemstatus
- Apache-Webseiten und APIs
- Gameserver
- Datenbankverwaltung
- NAS-Freigaben
- Backupstatus und Monitoring

Die Backupskripte laufen unabhängig von Homepage. Über eine kleine Statusdatei oder API kannst du den letzten erfolgreichen Backupzeitpunkt und Fehler in das Dashboard anzeigen.

## 8. Ergebnis

Einheitlichen Ubuntu-Server, der alle gewünschten Aufgaben übernimmt:

- NAS und Dateifreigaben mit Samba
- Gameserver und Webhosting
- Datenbanken und eigene C++-/C#-Anwendungen
- Docker für optionale Isolation
- Automatisierte Backups und Monitoring
- Zentrale Verwaltung über Homepage
