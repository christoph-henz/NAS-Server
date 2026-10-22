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
