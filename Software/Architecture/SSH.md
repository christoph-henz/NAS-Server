# SSH-Handling – Ubuntu Server

## 1. Grundkonzept

Jeder Benutzer erhält seine eigene SSH-Schlüsselkonfiguration. Die öffentlichen Schlüssel werden serverseitig in der Datei `authorized_keys` des jeweiligen Benutzers hinterlegt.

| Benutzer    | SSH-Zugriff        | Berechtigung              |
| ----------- | ------------------ | ------------------------- |
| `admin`     | Ja                 | Administration mit `sudo` |
| `christoph` | Ja, falls benötigt | Persönlicher Benutzer     |
| `julian`    | Ja, falls benötigt | Persönlicher Benutzer     |
| `games`     | Nein               | Technisches Dienstkonto   |
| `backup`    | Nein               | Technisches Dienstkonto   |

## 2. Verzeichnisstruktur

```
/home/
├── admin/
│   └── .ssh/
│       └── authorized_keys
├── christoph/
│   └── .ssh/
│       └── authorized_keys
└── julian/
    └── .ssh/
        └── authorized_keys
```

Jede `authorized_keys`-Datei enthält die öffentlichen Schlüssel, die sich als der jeweilige Benutzer anmelden dürfen.

Wichtig: Private Schlüssel bleiben auf den Client-Geräten. Sie werden nicht auf den Server kopiert.

## 3. Rechte auf dem Server

Für jeden Benutzer gelten folgende Berechtigungen:

| Pfad                              | Eigentümer          | Modus |
| --------------------------------- | ------------------- | ----- |
| `/home/USER/.ssh/`                | jeweiliger Benutzer | `700` |
| `/home/USER/.ssh/authorized_keys` | jeweiliger Benutzer | `600` |

Beispiel für `admin`:

```shell
sudo install -d -m 700 -o admin -g admin /home/admin/.ssh
sudo touch /home/admin/.ssh/authorized_keys
sudo chown admin:admin /home/admin/.ssh/authorized_keys
sudo chmod 600 /home/admin/.ssh/authorized_keys
```

Der öffentliche Schlüssel wird anschließend in `authorized_keys` eingefügt.

## 4. Ablauf zur Einrichtung

1. SSH-Schlüsselpaar auf dem Client erstellen

   `bash     ssh-keygen -t ed25519     `

   Den privaten Schlüssel sicher auf dem Client speichern und möglichst mit einer Passphrase schützen.
2. Öffentlichen Schlüssel auf den Server übertragen

   Beispielsweise mit `ssh-copy-id`, sofern die bisherige Anmeldung dies erlaubt:

   `bash     ssh-copy-id admin@SERVER-IP     `

   Alternativ den Inhalt der öffentlichen `.pub`-Datei in die `authorized_keys` des Zielbenutzers eintragen.
3. Anmeldung testen

   `bash     ssh admin@SERVER-IP     `

   Bei mehreren Schlüsseln oder Clients kann ein SSH-Clientprofil in `~/.ssh/config` hilfreich sein.
4. Erst danach die SSH-Sicherheit verschärfen

   Passwortanmeldung und Root-Login erst deaktivieren, wenn die Schlüsselanmeldung zuverlässig funktioniert und ein Wiederherstellungsweg vorhanden ist.

## 5. Zentrale Sicherheitsregeln

- SSH-Zugriff nur für Benutzer aktivieren, die ihn tatsächlich benötigen.
- `admin` ist das Administrationskonto; `sudo` nur gezielt vergeben.
- Keine gemeinsamen privaten Schlüssel für mehrere Personen verwenden.
- Den Zugriff über die SSH-Konfiguration unter `/etc/ssh/sshd_config` bzw. Dateien in `/etc/ssh/sshd_config.d/` verwalten.
- Nach Änderungen die Konfiguration prüfen und den Dienst kontrolliert neu laden:
  ```shell
  sudo sshd -t
  sudo systemctl reload ssh
  ```
- SSH nicht unnötig direkt aus dem Internet erreichbar machen; für externe Administration ist ein VPN meist die bessere Lösung.

Zuerst den SSH-Zugang für `admin` einrichten und in einem zweiten Terminal testen. Danach bei Bedarf separate SSH-Zugänge für `christoph` und `julian` ergänzen. Die NAS-Zugänge über Samba benötigen dagegen keine SSH-Anmeldung.