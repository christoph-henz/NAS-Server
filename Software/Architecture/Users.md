# Ubuntu-Server: Benutzer- und Gruppen- berechtigungskonzept

Für den Server mit NAS, Gameservern, Apache und automatisierten Backups wird für eine klare Trennung zwischen persönlichen Benutzerkonten und technischen Dienstkonten mehrere Nutzer angelegt.

Die folgende Struktur ist auf Ubuntu Server direkt auf der Hardware, einer SSD für das Betriebssystem und einer HDD für Daten ausgelegt.

## 1. Benutzer und Gruppen

| Konto    | Typ                         | Aufgaben                          | Rechte                                   |
| -------- | --------------------------- | --------------------------------- | ---------------------------------------- |
| `admin`  | Persönlicher Linux-Benutzer | Administration, Updates, SSH      | `sudo`                                   |
| `christoph` | Linux- und Samba-Benutzer   | NAS-Zugriff, private Dateien      | Kein `sudo`                              |
| `julian` | Linux- und Samba-Benutzer   | NAS-Zugriff, private Dateien      | Kein `sudo`                              |
| `games`  | Technisches Dienstkonto     | Aktive Gameserver und Spielstände | Kein `sudo`, keine interaktive Anmeldung |
| `backup` | Technisches Dienstkonto     | Automatisierte Backups            | Kein `sudo` für interaktive Nutzung      |

Gruppen:

- `nasusers`: gemeinsame NAS-Dateien für `christoph` und `julian`.
- `games`: Zugriff auf die aktiven Gameserver und auf die Gameserver-Backups.
- `backup`: Ausführung der Sicherungsjobs und gezielter Zugriff auf die jeweiligen Quellen und Ziele.

Die Gruppenmitgliedschaft allein gewährt noch keinen Zugriff: Eigentümer, Gruppenrechte, ACLs und gegebenenfalls Samba-Freigaberegeln müssen zusammenpassen.

## 2. Eigentümer und Berechtigungen


| Pfad                                    | Eigentümer:Gruppe | Modus    | Zweck                                        |
| --------------------------------------- | ----------------- | -------- | -------------------------------------------- |
| `/home/admin`                           | `admin:admin`     | `700`    | Private Admin-Dateien                        |
| `/srv/games`                            | `root:games`      | `2750`   | Gameserver-Verzeichnis                       |
| `/srv/games/minecraft`                  | `games:games`     | `2770`   | Aktive Minecraft-Dateien                     |
| `/srv/games/weitere-games`              | `games:games`     | `2770`   | Weitere aktive Gameserver                    |
| `/srv/data/shares/public`               | `root:nasusers`   | `2770`   | Gemeinsame NAS-Dateien                       |
| `/srv/data/shares/christoph`               | `christoph:christoph`   | `700`    | Private Dateien von Person 1                 |
| `/srv/data/shares/julian`               | `julian:julian`   | `700`    | Private Dateien von Person 2                 |
| `/srv/data/backups/system`              | `backup:backup`   | `700`    | System-Backups                               |
| `/srv/data/backups/games`               | `backup:backup`   | `750`    | Übergeordnetes Gameserver-Backup-Verzeichnis |
| `/srv/data/backups/games/minecraft`     | `backup:games`    | `2770`\* | Minecraft-Backups                            |
| `/srv/data/backups/games/weitere-games` | `backup:games`    | `2770`\* | Weitere Gameserver-Backups                   |
| `/srv/data/backups/databases`           | `backup:backup`   | `700`    | Datenbank-Backups                            |
| `/srv/data/media`                       | `root:nasusers`   | `2770`   | Gemeinsame Mediendateien                     |

\* Bei den Gameserver-Backups erlaubt `2770` auch der Gruppe `games` Schreibzugriff. Das entspricht dem Wunsch, dass `games` auf die Sicherungen zugreifen kann, erlaubt aber auch Änderungen und Löschungen. Wenn Backups vor Gameserver-Prozessen geschützt werden sollen, ist ein restriktiveres Berechtigungsmodell vorzuziehen.

Für Apache sollten Webinhalte nicht pauschal für alle Benutzer beschreibbar sein. Üblicherweise gehören sie `root` oder einem dedizierten Deployment-Konto; der Webserver benötigt nur die erforderlichen Leserechte. Verzeichnisse unter `/etc` bleiben grundsätzlich durch `root` geschützt.

## 3. Einrichten der Verzeichnisse

Vor der Ausführung muss die HDD korrekt unter `/srv/data` eingebunden sein. Muss geprüft werden!!:

```shell
findmnt /srv/data
df -h /srv/data
```


```shell
# Aktive Gameserver auf der SSD
sudo mkdir -p /srv/games/minecraft
sudo mkdir -p /srv/games/weitere-games

sudo chown root:games /srv/games
sudo chmod 2750 /srv/games

sudo chown -R games:games /srv/games/minecraft
sudo chmod 2770 /srv/games/minecraft

sudo chown -R games:games /srv/games/weitere-games
sudo chmod 2770 /srv/games/weitere-games

# NAS-Verzeichnisse auf der HDD
sudo mkdir -p /srv/data/shares/{public,christoph,julian}
sudo mkdir -p /srv/data/media

sudo chown root:nasusers /srv/data/shares/public
sudo chmod 2770 /srv/data/shares/public

sudo chown christoph:christoph /srv/data/shares/christoph
sudo chmod 700 /srv/data/shares/christoph

sudo chown julian:julian /srv/data/shares/julian
sudo chmod 700 /srv/data/shares/julian

sudo chown root:nasusers /srv/data/media
sudo chmod 2770 /srv/data/media

# Backup-Verzeichnisse auf der HDD
sudo mkdir -p /srv/data/backups/{system,games,databases}
sudo mkdir -p /srv/data/backups/games/{minecraft,weitere-games}

sudo chown backup:backup /srv/data/backups/system
sudo chmod 700 /srv/data/backups/system

sudo chown backup:backup /srv/data/backups/games
sudo chmod 750 /srv/data/backups/games

sudo chown backup:backup /srv/data/backups/databases
sudo chmod 700 /srv/data/backups/databases

sudo chown backup:games /srv/data/backups/games/minecraft
sudo chmod 2770 /srv/data/backups/games/minecraft

sudo chown backup:games /srv/data/backups/games/weitere-games
sudo chmod 2770 /srv/data/backups/games/weitere-games
```

Diese Befehle legen die Verzeichnisse und Rechte an. Sie konfigurieren noch nicht automatisch Samba, systemd, die Backup-Jobs oder die dauerhafte HDD-Einbindung. Falls bereits Daten in den Verzeichnissen liegen, sollten vorhandene Eigentümer und Berechtigungen vor Änderungen geprüft werden.

## 4. Zugriff des Backup-Kontos

Damit `backup` aktive Gameserver auf der SSD sichern kann, benötigt es Leserechte auf `/srv/games/...`. Das lässt sich über gezielte ACLs lösen, ohne `backup` zum Mitglied der Gruppe `games` machen zu müssen:

```shell
sudo apt install acl

sudo setfacl -R -m u:backup:rX /srv/games
sudo setfacl -R -d -m u:backup:rX /srv/games
```

Damit erhält `backup` Lese- und Verzeichniszugriffsrechte auf die vorhandenen Dateien und passende Standard-ACLs für neu erstellte Dateien. Die konkreten ACLs sollten anschließend mit `getfacl` geprüft werden.

Der Backup-Job muss außerdem konsistente Spielstände erzeugen – bei Minecraft beispielsweise durch einen kontrollierten Save-/Stop-Ablauf oder eine geeignete Backup-Schnittstelle.

## 5. Was zusätzlich konfiguriert werden muss

Die Linux-Dateirechte allein reichen noch nicht für den vollständigen Betrieb:

1. Samba: Freigaben für `public`, `christoph` und `julian` in `/etc/samba/smb.conf` definieren. Für `christoph` und `julian` müssen die privaten Freigaben jeweils auf den passenden Benutzer beschränkt sein.
2. Samba-Passwörter: Für beide NAS-Benutzer separat setzen, zum Beispiel mit `sudo smbpasswd -a christoph` und `sudo smbpasswd -a julian`. Samba-Konten setzen entsprechende Linux-Konten voraus.
3. Gameserver: Dienstdateien unter `/etc/systemd/system/` anlegen und den Dienst mit `User=games` sowie `Group=games` betreiben.
4. Backups: Einen systemd-Timer oder Cronjob einrichten und die Leserechte auf die zu sichernden Daten gezielt vergeben.
5. HDD-Mount: Die Festplatte über ihre UUID in `/etc/fstab` dauerhaft einbinden und den Mount vor dem ersten Schreiben testen.

Damit habt ihr eine saubere Trennung zwischen Administration, persönlichen NAS-Daten, Gameserver-Prozessen und Backup-Aufgaben. Die nächste sinnvolle Umsetzung wäre, die Samba-Freigaben samt Zugriffsschutz passend zu genau dieser Struktur einzurichten.