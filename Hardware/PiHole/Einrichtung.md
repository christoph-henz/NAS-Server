# Pi-hole auf dem Ubuntu-NAS-/Server einrichten

Pi Hole wird direkt auf Ubuntu Server installiert. Der Router von Magenta Telekom bleibt zunächst für Internetzugang, Gateway und DHCP zuständig. Pi-hole übernimmt die DNS-Filterung für das gesamte Heimnetzwerk.

!!! Die Konfiguration muss einwandfrei funktionieren mit: Samba, Apache, Backupverwaltung, Gameserver und Architekturtests.

Klare Aufgabentrennung: 
Der Router verteilt IP-Adressen, Pi-hole filtert DNS-Anfragen (und Apache stellt eigenen Webanwendungen bereit).

## Zielarchitektur


## 1. IP-Plan für den Server

Die private IPv4-Netzadresse ist `192.168.56.1`.

| Komponente            | Beispiel                   | Aufgabe                              |
| --------------------- | -------------------------- | ------------------------------------ |
| Router                | `192.168.56.1`             | Gateway und DHCP                     |
| Ubuntu-Server         | `192.168.56.2`             | Feste Adresse für alle Serverdienste |
| DHCP-Adresspool       | `192.168.56.100–199`       | Dynamische Adressen für Clients      |
| Pi-hole DNS           | `192.168.56.2:53`          | DNS für alle Geräte                  |
| Pi-hole Weboberfläche | `http://192.168.56.2/admin` | Administration (evtl. übwe Homepage) |
| Apache                | `http://192.168.56.2:8080`  | Eigene Webanwendungen                |

Pi-hole und Apache teilen sich die Server-IP, aber unterschiedliche Ports. Der DNS-Dienst benötigt TCP und UDP auf Port 53. Die Pi-hole-Weboberfläche verwendet standardmäßig HTTP-Port 80; falls dieser durch Apache belegt ist, muss die Portbelegung gezielt konfiguriert werden.&#x20;

[Documentation](https://docs.pi-hole.net/main/prerequisites/?h=webserver)

Pi-hole documentation



### Zuerst das tatsächliche Netzwerk ermitteln

```shell
ip -4 addr
ip route
resolvectl status
```

Damit sieht man:

- welche IP-Adresse der Server aktuell hat,
- welches Gateway der Router verwendet,
- welches Netzwerk und welche Netzmaske aktiv sind,
- welche DNS-Server Ubuntu momentan nutzt.


```shell
sudo ss -lntup | grep -E ':(53|80|443|8080)\b'
```

(überprüft die Portbelegung)

## 2. Pi-hole nativ installieren

Die offizielle Pi-hole-Installation unterstützt Ubuntu. Für den Server Installation ohne Docker :) .&#x20;

[Documentation](https://docs.pi-hole.net/main/prerequisites/?h=webserver)

Pi-hole documentation

Vor der Installation:

1. Ubuntu-Version mit `lsb_release -a` prüfen.
2. Feste IP beziehungsweise DHCP-Reservierung einrichten (?).
3. Ubuntu aktualisieren.
4. Portbelegung kontrollieren.

```shell
sudo apt update
sudo apt upgrade
sudo apt install curl wget dnsutils
```

Danach offiziellen Installer herunterladen, prüfen und starten:

```shell
wget -O basic-install.sh https://install.pi-hole.net
less basic-install.sh
sudo bash basic-install.sh
```

Während der Installation:

- Netzwerkschnittstelle: die kabelgebundene LAN-Schnittstelle, normalerweise `enp...`
- IP-Konfiguration: die zuvor reservierte beziehungsweise statische Server-IP
- Upstream-DNS: zunächst beispielsweise Cloudflare oder Quad9
- Blocklisten: die vorgeschlagenen Standardlisten
- Weboberfläche: aktivieren
- Webserver: aktivieren, sofern die Portkonfiguration mit Apache geklärt ist

## 3. Apache und Pi-hole sauber trennen

Folgende Portverteilungen sind sinnvoll:

| Dienst                | Port       | Zweck                       |
| --------------------- | ---------- | --------------------------- |
| Pi-hole DNS           | TCP/UDP 53 | DNS-Auflösung und Filterung |
| Pi-hole Weboberfläche | TCP 80     | Administration              |
| Apache                | TCP 8080   | Eigene Websites und Tests   |
| SSH                   | TCP 22     | Fernadministration          |

Damit bleibt Pi-hole über `http://SERVER-IP/admin` erreichbar, während Apache über `http://SERVER-IP:8080` läuft.

### Apache auf Port 8080 umstellen

Konfiguration prüfen:

```shell
sudo grep -RniE '^[[:space:]]*Listen|<VirtualHost' \
  /etc/apache2/ports.conf \
  /etc/apache2/sites-enabled/
```

Wenn Apache bisher Port 80 verwendet, muss in `/etc/apache2/ports.conf` der passende `Listen`-Eintrag auf `8080` geändern werden und die aktiven VirtualHosts von `*:80` auf `*:8080` umgestellt werden.

Anschließend:

```shell
sudo apache2ctl configtest
sudo systemctl reload apache2
sudo ss -lntup | grep -E ':(80|8080)\b'
```

Ziel: Pi-hole belegt Port 80 und Apache Port 8080.

## 4. Router von Magenta Telekom konfigurieren

Feste Adresse für Ubuntuserver festlegen

Wenn manuelle Zuweisung fehlschlägt: DHCP-Server von Pi-hole übernehmen lassen. 


## 5. IPv6: Damit nicht die Hälfte der Geräte Pi-hole umgeht

IPv6 nicht pauschal deaktivieren!!!

Zielzustand ist:

- IPv4-DNS zeigt auf die IPv4-Adresse von Pi-hole.
- IPv6-DNS zeigt auf eine dauerhaft erreichbare IPv6-Adresse von Pi-hole, sofern der Router diese Einstellung unterstützt.
- Router Advertisements (RA) und DHCPv6 verteilen keine alternative DNS-Adresse, die die Filterung umgeht.

Wenn der Router weiterhin sich selbst als IPv6-DNS ankündigt, können Geräte Pi-hole teilweise umgehen. 
Das kann auch passieren, wenn IPv4 bereits korrekt konfiguriert ist. 


Prüfen nach der Einrichtung:

- Windows: `ipconfig /all`
- Ubuntu: `resolvectl status`

## 6. Firewall mit UFW absichern

Pi-hole soll aus dem LAN erreichbar sein, aber nicht ungeschützt aus dem Internet. 
Die Pi-hole-Dokumentation beschreibt die notwendigen Ports und weist darauf hin, Firewall-Regeln an das tatsächlich verwendete lokale Netz anzupassen.&#x20;


Wenn das LAN `192.168.56.0` verwendet, wären die folgenden Regeln sinnvoll:

```shell
sudo ufw allow from 192.168.1.0/24 to any port 53 proto udp
sudo ufw allow from 192.168.1.0/24 to any port 53 proto tcp

sudo ufw allow from 192.168.1.0/24 to any port 80 proto tcp
sudo ufw allow from 192.168.1.0/24 to any port 8080 proto tcp
```

Für SSH über das LAN, muss auch der SSH-Zugriff erlaubt sein. Vor einer Aktivierung der Firewall sollte unbedingt die bestehende SSH-Verbindung berücksichtigt und sicherstellen, dass sie sich nicht aussperrt.

```shell
sudo ufw status verbose
```
