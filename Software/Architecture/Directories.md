# Ubuntu-Server: Pfadkonzept

## 1. Verzeichnisstruktur der Festplatten

SSD – Betriebssystem und Anwendungen

System, Programme, Logs und Konfigurationen

```
/
├── etc/
│   ├── samba/
│   │   └── smb.conf                 # Samba-Konfiguration
│   ├── apache2/                     # Apache-Konfiguration
│   ├── systemd/
│   │   └── system/                  # Eigene systemd-Dienste und Timer
│   └── ...
├── home/
│   ├── admin/                       # Persönliches Admin-Verzeichnis
│   │   └── .ssh/
│   ├── christoph/                   # Persönliches Verzeichnis
│   │   └── .ssh/
│   └── julian/                      # Persönliches Verzeichnis
│       └── .ssh/
├── srv/
│   └── games/                       # Aktive Gameserver auf SSD
│       ├── minecraft/
│       │   └── modpacks/
│       │       ├── modpack1/
│       │       │   ├── (.v1/)
│       │       │   ├── (.v2/)
│       │       │   ├── v3/
│       │       │   └── world/
│       │       ├── modpack2/
│       │       │   ├── (.v1/)
│       │       │   ├── (.v2/)
│       │       │   ├── v3/
│       │       │   └── world/
│       │       └── modpack3/
│       │           ├── (.v1/)
│       │           ├── (.v2/)
│       │           ├── v3/
│       │           └── world/
│       ├── weitere-games1/
│       ├── weitere-games2/
│       └── weitere-games3/
├── var/
│   ├── log/                         # System- und Dienstprotokolle
│   └── www/
│       └── html/                    # Apache-Standard-Webverzeichnis
├── opt/                             # Optionale Zusatzsoftware
└── ...
```

HDD – Nutzdaten

Dauerhaft unter `/srv/data` eingebunden

```
/srv/data/                           # Einhängepunkt der HDD
├── shares/
│   ├── public/                      # Gemeinsame Dateien
│   ├── gamer1/                      # Private Dateien Person 1
│   └── gamer2/                      # Private Dateien Person 2
├── backups/
│   ├── system/                      # System- und Konfigurationssicherungen
│   ├── games/
│   │   ├── minecraft/               # Minecraft-Backups
│   │   └── weitere-games/           # Backups weiterer Gameserver
│   └── databases/                   # Datenbank-Dumps
└── media/                           # Optional: Bilder, Videos und Musik
```

Die Verzeichnisse sind ein Vorschlag, keine automatisch vorhandene Ubuntu-Struktur. `/srv/data` muss nach dem Einrichten der HDD dauerhaft über `/etc/fstab` eingebunden werden. Wenn möglich, werden Datenbanken und stark beanspruchte Anwendungen auf der SSD gehalten; große Spielstände und Backups können auf die HDD.

## 2. Eigentümer und Berechtigungen

Die folgende Tabelle zeigt die empfohlenen Standardrechte. Die Verzeichnisse müssen entsprechend angelegt und konfiguriert werden.

| Pfad                                    | Eigentümer:Gruppe | Modus    | Zweck                                        |
| --------------------------------------- | ----------------- | -------- | -------------------------------------------- |
| `/home/admin`                           | `admin:admin`     | `700`    | Private Admin-Dateien                        |
| `/srv/games`                            | `root:games`      | `2750`   | Gameserver-Verzeichnis                       |
| `/srv/games/minecraft`                  | `games:games`     | `2770`   | Aktive Minecraft-Dateien                     |
| `/srv/games/weitere-games`              | `games:games`     | `2770`   | Weitere aktive Gameserver                    |
| `/srv/data/shares/public`               | `root:nasusers`   | `2770`   | Gemeinsame NAS-Dateien                       |
| `/srv/data/shares/christoph`            | `christoph:christoph`   | `700`    | Private Dateien von Person 1                 |
| `/srv/data/shares/julian`               | `julian:julian`   | `700`    | Private Dateien von Person 2                 |
| `/srv/data/backups/system`              | `backup:backup`   | `700`    | System-Backups                               |
| `/srv/data/backups/games`               | `backup:backup`   | `750`    | Übergeordnetes Gameserver-Backup-Verzeichnis |
| `/srv/data/backups/games/minecraft`     | `backup:games`    | `2770`\* | Minecraft-Backups                            |
| `/srv/data/backups/games/weitere-games` | `backup:games`    | `2770`\* | Weitere Gameserver-Backups                   |
| `/srv/data/backups/databases`           | `backup:backup`   | `700`    | Datenbank-Backups                            |
| `/srv/data/media`                       | `root:nasusers`   | `2770`   | Gemeinsame Mediendateien                     |

\* Bei den Gameserver-Backups erlaubt `2770` auch der Gruppe `games` Schreibzugriff. Das entspricht dem Wunsch, dass `games` auf die Sicherungen zugreifen kann, erlaubt aber auch Änderungen und Löschungen. Wenn Backups vor Gameserver-Prozessen geschützt werden sollen, ist ein restriktiveres Berechtigungsmodell vorzuziehen.

Für Apache sollten Webinhalte nicht pauschal für alle Benutzer beschreibbar sein. Üblicherweise gehören sie `root` oder einem dedizierten Deployment-Konto; der Webserver benötigt nur die erforderlichen Leserechte. Verzeichnisse unter `/etc` bleiben grundsätzlich durch `root` geschützt.                                          |

Was bedeuten die Rechte?

- `700`: Nur der Eigentümer darf lesen, schreiben und Verzeichnisse betreten.
- `750`: Eigentümer darf alles, Gruppenmitglieder dürfen lesen und betreten.
- `2770`: Eigentümer und Gruppe dürfen alles; das führende `2` sorgt dafür, dass neue Dateien und Verzeichnisse die Gruppe des übergeordneten Verzeichnisses übernehmen.
- `root` bleibt Eigentümer wichtiger Systemkonfigurationen, damit normale Benutzer diese nicht verändern können.

Bei privaten NAS-Verzeichnissen muss Samba ebenfalls so konfiguriert werden, dass die Freigabe nur dem jeweiligen Benutzer zugänglich ist.