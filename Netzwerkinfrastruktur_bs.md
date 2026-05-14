# Technische Dokumentation: Externer Zugriff auf Proxmox-VMs über pfSense, NAT und DNS

<p align="center">
  <img src="/images/proxmox-img.webp" alt="Proxmox" width="20%">
  <img src="/images/pfsense-img.png" alt="pfSense" width="20%">
  <img src="/images/tailscale-img.png" alt="Tailscale" width="20%">
</p>

## 1. Ausgangslage

Im Rahmen des Projekts soll eine Netzwerkumgebung aufgebaut werden, in welcher zwei Ubuntu-Server-VMs in voneinander getrennten Netzen betrieben werden. Zusätzlich soll ein externer Zugriff auf diese Systeme ermöglicht werden, ohne direkt mit der öffentlichen IP-Adresse arbeiten zu müssen.

Die Umgebung basiert auf einem Proxmox-Host, auf welchem mehrere virtuelle Maschinen betrieben werden. Als zentrale Netzwerkkomponente wird eine pfSense-Firewall eingesetzt. Diese übernimmt die Funktion des Gateways zwischen den internen Netzen und dem WAN-Netz.

Eine Anforderung war, dass ein Gateway mit zwei internen Netzen bereitgestellt wird:

- Netz 1: `10.10.10.0/24` mit Gateway `10.10.10.1`
- Netz 2: `10.10.20.0/24` mit Gateway `10.10.20.1`

In diesen beiden Netzen befindet sich jeweils eine eigene Ubuntu-Server-VM. Die VMs dienen dazu, die Netztrennung sowie die Kommunikation zwischen den beiden Netzen zu testen. Zusätzlich können die Systeme später für Dienste wie Git oder Ansible verwendet werden.

## 2. Ziel der Umsetzung

Ziel des Projekts ist der Aufbau einer funktionierenden virtuellen Netzwerkumgebung mit folgenden Eigenschaften:

- Aufbau von zwei getrennten internen Netzbereichen
- Einsatz einer pfSense-VM als Firewall und Gateway
- Routing zwischen WAN, LAN und OPT1 über pfSense
- Zugriff auf die internen VMs über Portweiterleitungen
- Zugriff von extern über die öffentliche IP-Adresse des Routers
- Nutzung eines DNS-Namens über Cloudflare, damit nicht direkt mit der öffentlichen IP-Adresse gearbeitet werden muss
- Automatische Aktualisierung der öffentlichen IP-Adresse über Dynamic DNS
- Separater Administrationszugriff über Tailscale

## 3. Netzwerkübersicht

Die Umgebung wurde auf einem Proxmox-Host aufgebaut. Neben der bestehenden Standard-Bridge `vmbr0` wurden zwei zusätzliche Bridges erstellt.

| Komponente | Funktion | Zugewiesenes Netz / Interface |
|---|---|---|
| Proxmox `vmbr0` | WAN-Anbindung der pfSense | Externes / bestehendes Netzwerk |
| Proxmox `vmbr1` | Internes Netz für VM im ersten Netz | `10.10.10.0/24` |
| Proxmox `vmbr2` | Internes Netz für VM im zweiten Netz | `10.10.20.0/24` |
| pfSense WAN | Verbindung Richtung Router / Internet | `192.168.1.15` über `vmbr0` |
| pfSense LAN | Gateway für Netz 1 | `10.10.10.1` |
| pfSense OPT1 | Gateway für Netz 2 | `10.10.20.1` |
| VM im ersten Netz | Ubuntu Server | `10.10.10.11` |
| VM im zweiten Netz | Ubuntu Server | `10.10.20.11` |
| Ubuntu Desktop VM | Administrations-Client | Zugriff auf pfSense WebGUI |

## 3.1 Beschreibung der Netzwerkinfrastruktur

<p align="center">
  <img src="/images/netzwerkinfrastruktur_bs_admin.drawio.png" alt="Netzwerkinfrastruktur BS Admin" width="60%">
</p>

Die Zeichnung zeigt die Netzwerkinfrastruktur für den externen Zugriff auf zwei virtuelle Maschinen. Die Umgebung besteht aus einem Proxmox-Server, einer pfSense-Firewall, einem UniFi-Router sowie DNS über die Domain `athena-forge.ch`.

Der Zugriff von extern erfolgt über den DNS-Namen `pfsense-bs.athena-forge.ch`. Dieser Name zeigt auf die öffentliche IP-Adresse des Routers. Damit die öffentliche IP-Adresse nicht manuell angepasst werden muss, wird Dynamic DNS verwendet. Die DNS-Verwaltung erfolgt über Cloudflare. Der UniFi-Router aktualisiert den DNS-Eintrag automatisch, falls sich die öffentliche IP-Adresse ändert.

Die Domain `athena-forge.ch` ist bei Hostpoint registriert. Die Nameserver der Domain wurden jedoch auf Cloudflare geändert. Dadurch wird die DNS-Zone nicht mehr bei Hostpoint, sondern bei Cloudflare verwaltet.

Vom Internet aus erreichen die Verbindungen zuerst den UniFi-Router. Dort werden die externen Ports `2222` und `2223` an die WAN-Adresse der pfSense weitergeleitet. Die pfSense hat auf der WAN-Seite die IP-Adresse `192.168.1.15`.

Die pfSense übernimmt anschliessend die Firewall- und Routing-Funktion. Zusätzlich führt sie NAT-Regeln aus, damit die eingehenden Verbindungen an die richtigen internen VMs weitergeleitet werden.

Die erste VM befindet sich im Netz `10.10.10.0/24` und hat die IP-Adresse `10.10.10.11`. Sie ist von extern über folgenden Befehl erreichbar:

```bash
ssh <benutzer>@pfsense-bs.athena-forge.ch -p 2222
```

Die zweite VM befindet sich im Netz `10.10.20.0/24` und hat die IP-Adresse `10.10.20.11`. Sie ist von extern über folgenden Befehl erreichbar:

```bash
ssh <benutzer>@pfsense-bs.athena-forge.ch -p 2223
```

Auf den VMs selbst läuft SSH weiterhin auf dem Standard-Port `22`. Die Ports `2222` und `2223` sind nur die externen Ports. Die Übersetzung auf Port `22` erfolgt durch die NAT-Regeln auf der pfSense.

Die beiden internen Netze sind voneinander getrennt. Das erste Netz läuft über das LAN-Interface der pfSense, das zweite Netz über das OPT1-Interface. Dadurch kann getestet werden, ob zwei getrennte Netze über ein Gateway korrekt betrieben und kontrolliert miteinander verbunden werden können.

Zusätzlich ist in der Zeichnung Tailscale eingezeichnet. Tailscale dient als separater Administrationszugang. Administratoren können sich über Tailscale sicher mit der Umgebung verbinden, ohne dafür zusätzliche öffentliche Ports freigeben zu müssen. Der normale Benutzerzugriff läuft über DNS, UniFi-Portweiterleitung und pfSense-NAT. Der administrative Zugriff kann getrennt davon über Tailscale erfolgen.

Zusammengefasst zeigt die Zeichnung zwei Zugriffswege:

| Zugriff | Zweck |
|---|---|
| Cloudflare DNS → UniFi → pfSense → VM | Zugriff für Benutzer oder Kunden auf freigegebene VMs |
| Tailscale → interne Infrastruktur | Sicherer administrativer Zugriff für Betreiber |

## 4. Aufbau auf dem Proxmox-Host

### 4.1 Erstellung zusätzlicher Bridges

Auf dem Proxmox-Host existierte bereits die Bridge `vmbr0`. Diese wurde weiterhin für die WAN-Seite der pfSense verwendet.

Zusätzlich wurden zwei weitere Bridges erstellt:

- `vmbr1` für das interne Netz `10.10.10.0/24`
- `vmbr2` für das interne Netz `10.10.20.0/24`

Diese Bridges dienen als virtuelle Switches innerhalb von Proxmox. Dadurch können die VMs voneinander getrennt, aber kontrolliert über pfSense miteinander verbunden werden.

### 4.2 Erstellung der pfSense-VM

Anschliessend wurde eine pfSense-VM erstellt. Dieser VM wurden drei virtuelle Netzwerkadapter zugewiesen:

| pfSense Interface | Proxmox Bridge | Zweck |
|---|---|---|
| WAN | `vmbr0` | Verbindung zum bestehenden Netzwerk / Router, statische IP `192.168.1.15` |
| LAN | `vmbr1` | Internes Netz 1 |
| OPT1 | `vmbr2` | Internes Netz 2 |

Die pfSense übernimmt damit die Rolle des zentralen Gateways für beide internen Netze.

## 5. Konfiguration der pfSense

### 5.1 Interface-Zuweisung

Nach der Installation von pfSense wurden die Interfaces wie folgt zugewiesen:

- `vmbr0` als WAN
- `vmbr1` als LAN
- `vmbr2` als OPT1

Das LAN-Interface wurde für das Netz `10.10.10.0/24` verwendet. Das OPT1-Interface wurde für das Netz `10.10.20.0/24` verwendet.

Die Gateway-Adressen wurden wie folgt definiert:

| Interface | IP-Adresse | Netz |
|---|---:|---|
| WAN | `192.168.1.15` | bestehendes Router-Netz |
| LAN | `10.10.10.1` | `10.10.10.0/24` |
| OPT1 | `10.10.20.1` | `10.10.20.0/24` |

### 5.2 Aktivierung von OPT1

Damit Geräte im OPT1-Netz kommunizieren können, musste auf pfSense eine Firewall-Regel für OPT1 erstellt werden.

Dazu wurde über die pfSense-Konsole eine EasyRule erstellt:

```bash
easyrule pass opt1 any 10.10.20.0/24 any
```

Diese Regel erlaubt grundsätzlich Traffic aus dem OPT1-Netz. Ohne eine entsprechende Regel würde pfSense den Verkehr auf OPT1 standardmässig blockieren.

## 6. Erstellung der Ubuntu-VMs

Im nächsten Schritt wurden zwei Ubuntu-Server-VMs erstellt.

### 6.1 VM im ersten Netz

Die erste Ubuntu-VM wurde dem Netzwerk `vmbr1` zugewiesen. Dadurch befindet sie sich im Netz `10.10.10.0/24`.

| Einstellung | Wert |
|---|---|
| IP-Adresse | `10.10.10.11` |
| Netz | `10.10.10.0/24` |
| Gateway | `10.10.10.1` |
| Bridge | `vmbr1` |
| Distribution | Ubuntu Server |
| Zweck | Test-VM im ersten Netz, optional Git-Server |

### 6.2 VM im zweiten Netz

Die zweite Ubuntu-VM wurde dem Netzwerk `vmbr2` zugewiesen. Dadurch befindet sie sich im Netz `10.10.20.0/24`.

| Einstellung | Wert |
|---|---|
| IP-Adresse | `10.10.20.11` |
| Netz | `10.10.20.0/24` |
| Gateway | `10.10.20.1` |
| Bridge | `vmbr2` |
| Distribution | Ubuntu Server |
| Zweck | Test-VM im zweiten Netz, optional Ansible-Server |

## 7. Administrationszugriff auf pfSense

Da die pfSense Weboberfläche aus Sicherheitsgründen nicht über das WAN-Interface erreichbar ist, wurde zusätzlich eine Ubuntu Desktop VM erstellt.

Diese Desktop-VM dient als Administrationssystem innerhalb der virtuellen Umgebung. Über diese VM kann die pfSense Weboberfläche erreicht und verwaltet werden.

Der Zugriff erfolgt intern über das jeweilige LAN-Interface der pfSense, zum Beispiel über:

```text
https://10.10.10.1
```

Damit ist die Administration der Firewall weiterhin möglich, ohne die pfSense WebGUI direkt nach aussen zu öffnen.

Zusätzlich kann der administrative Zugriff über Tailscale erfolgen. Dadurch können Administratoren sicher auf die Umgebung zugreifen, ohne die pfSense Weboberfläche öffentlich erreichbar zu machen.

## 8. NAT- und Portweiterleitung auf pfSense

Damit von extern auf die internen Ubuntu-Server zugegriffen werden kann, wurden auf der pfSense NAT-Portweiterleitungen eingerichtet.

Die Portweiterleitung wurde auf dem WAN-Interface der pfSense erstellt. Als Protokoll wurde TCP verwendet. Die Zieladresse ist jeweils die WAN-Adresse der pfSense. Der eingehende Port wird anschliessend auf den SSH-Port der jeweiligen internen VM weitergeleitet.

### 8.1 NAT-Regel für die VM im zweiten Netz

Eine der NAT-Regeln leitet den externen Port `2223` auf die interne IP-Adresse `10.10.20.11` weiter. Als Ziel-Port wurde SSH verwendet.

| Einstellung | Wert |
|---|---|
| Interface | WAN |
| Address Family | IPv4 |
| Protocol | TCP |
| Destination | WAN address |
| Destination Port | `2223` |
| Redirect Target IP | `10.10.20.11` |
| Redirect Target Port | SSH / `22` |

Damit kann ein SSH-Zugriff auf die VM im zweiten Netz über den Port `2223` erfolgen.

Beispiel:

```bash
ssh <benutzername>@pfsense-bs.athena-forge.ch -p 2223
```

### 8.2 NAT-Regel für die VM im ersten Netz

Zusätzlich wurde eine zweite NAT-Regel erstellt. Diese verwendet den externen Port `2222` und leitet auf den SSH-Port der VM im ersten Netz mit der IP-Adresse `10.10.10.11` weiter.

| Externer Port | Interne Ziel-VM | Interner Port | Zweck |
|---:|---|---:|---|
| `2222` | VM im ersten Netz `10.10.10.11` | `22` | SSH-Zugriff auf VM im ersten Netz |
| `2223` | VM im zweiten Netz `10.10.20.11` | `22` | SSH-Zugriff auf VM im zweiten Netz |

Dadurch können zwei verschiedene interne Server über unterschiedliche externe Ports erreicht werden.

## 9. Portweiterleitung auf dem UniFi-Router

Da sich die pfSense hinter dem UniFi-Router befindet, musste zusätzlich auf dem Router eine Portweiterleitung eingerichtet werden.

Auf dem UniFi-Router wurden die Ports `2222` bis `2223` geöffnet und an die statische WAN-Adresse der pfSense `192.168.1.15` weitergeleitet.

Dadurch ergibt sich folgender Verbindungsweg:

```text
Internet
  ↓
Öffentliche IP-Adresse des Routers
  ↓
UniFi-Portweiterleitung 2222-2223
  ↓
pfSense WAN 192.168.1.15
  ↓
pfSense NAT-Regel
  ↓
Interne Ubuntu-VM
```

Ohne diese Portweiterleitung auf dem UniFi-Router würden eingehende Verbindungen aus dem Internet die pfSense nicht erreichen.

## 10. DNS-Verwaltung über Cloudflare

Damit der Zugriff nicht über die öffentliche IP-Adresse erfolgen muss, wird ein DNS-Name verwendet.

Der DNS-Name lautet:

```text
pfsense-bs.athena-forge.ch
```

Die Domain `athena-forge.ch` ist bei Hostpoint registriert. Die Nameserver der Domain wurden jedoch auf Cloudflare geändert. Dadurch wird die DNS-Zone nicht mehr bei Hostpoint, sondern bei Cloudflare verwaltet.

In Cloudflare ist für `pfsense-bs.athena-forge.ch` ein DNS-Eintrag eingerichtet. Dieser zeigt auf die öffentliche IP-Adresse des UniFi-Routers.

Damit die öffentliche IP-Adresse bei einer Änderung nicht manuell angepasst werden muss, wird Dynamic DNS verwendet. Der UniFi-Router aktualisiert den DNS-Eintrag bei Cloudflare automatisch.

Wichtig ist, dass der DNS-Eintrag in Cloudflare auf `DNS only` gesetzt ist. Der Cloudflare Proxy wird nicht verwendet, da der Zugriff über SSH und eigene Ports erfolgt.

Beispiel für den Zugriff per SSH:

```bash
ssh <benutzername>@pfsense-bs.athena-forge.ch -p 2222  # VM im ersten Netz
ssh <benutzername>@pfsense-bs.athena-forge.ch -p 2223  # VM im zweiten Netz
```

## 11. Gesamtablauf des externen Zugriffs

Der externe Zugriff funktioniert in mehreren Schritten:

1. Der Client verbindet sich mit `pfsense-bs.athena-forge.ch` auf Port `2222` für die VM im ersten Netz oder auf Port `2223` für die VM im zweiten Netz.
2. Der DNS-Name wird über Cloudflare zur öffentlichen IP-Adresse des UniFi-Routers aufgelöst.
3. Der UniFi-Router nimmt die Verbindung entgegen und leitet sie an die WAN-Adresse der pfSense `192.168.1.15` weiter.
4. pfSense verarbeitet die eingehende Verbindung auf dem WAN-Interface.
5. Die NAT-Regel der pfSense leitet die Verbindung an die passende interne Ubuntu-VM weiter.
6. Die Ziel-VM nimmt die Verbindung auf Port `22` entgegen.

## 12. Sicherheitsaspekte

Bei dieser Umsetzung wird SSH aus dem Internet erreichbar gemacht. Deshalb sind einige Sicherheitsmassnahmen wichtig:

- SSH-Zugriff sollte möglichst nur mit SSH-Key und nicht mit Passwort erlaubt werden.
- Root-Login per SSH sollte deaktiviert sein.
- Es sollten nur die benötigten Ports freigegeben werden.
- Die Portweiterleitungen sollten dokumentiert und regelmässig überprüft werden.
- Auf pfSense sollten keine unnötigen WAN-Regeln erstellt werden.
- Die pfSense WebGUI sollte nicht direkt über das WAN erreichbar sein.
- Der Cloudflare Proxy bleibt für SSH deaktiviert, da der DNS-Eintrag nur für die Namensauflösung verwendet wird.
- Der administrative Zugriff sollte bevorzugt über Tailscale erfolgen, damit keine zusätzlichen Administrationsports öffentlich geöffnet werden müssen.

## 13. Ergebnis

Durch die Umsetzung wurde eine virtuelle Netzwerkumgebung auf Proxmox aufgebaut, welche zwei getrennte interne Netze über eine pfSense-Firewall bereitstellt.

Die beiden Ubuntu-VMs befinden sich in unterschiedlichen Netzbereichen und verwenden jeweils pfSense als Gateway. Über NAT-Regeln auf der pfSense sowie eine zusätzliche Portweiterleitung auf dem UniFi-Router können die Systeme von extern erreicht werden.

Der Zugriff erfolgt nicht direkt über die öffentliche IP-Adresse, sondern über den DNS-Namen `pfsense-bs.athena-forge.ch`, welcher in Cloudflare verwaltet wird. Dadurch ist der Zugriff einfacher und verständlicher. Zusätzlich wird die öffentliche IP-Adresse über Dynamic DNS automatisch aktualisiert.

Die pfSense Weboberfläche bleibt intern erreichbar und wird nicht direkt ins Internet veröffentlicht. Für die Administration wurde eine separate Ubuntu Desktop VM erstellt, welche Zugriff auf die interne pfSense WebGUI ermöglicht. Zusätzlich kann Tailscale als sicherer Administrationszugang verwendet werden.

## 14. Zusammenfassung der wichtigsten Adressen und Ports

| Element | Wert |
|---|---|
| pfSense WAN-IP | `192.168.1.15` |
| pfSense LAN-Gateway | `10.10.10.1` |
| pfSense OPT1-Gateway | `10.10.20.1` |
| VM im ersten Netz | `10.10.10.11` |
| VM im zweiten Netz | `10.10.20.11` |
| Externer DNS-Name | `pfsense-bs.athena-forge.ch` |
| SSH VM im ersten Netz | Port `2222` |
| SSH VM im zweiten Netz | Port `2223` |
| DNS-Verwaltung | Cloudflare |
| Domain-Registrar | Hostpoint |
| Administrationszugriff | Tailscale / interne Ubuntu Desktop VM |

## 15. Kurzes Fazit

Das Projekt zeigt, wie mit Proxmox, pfSense, NAT, Cloudflare DNS und Tailscale eine getrennte virtuelle Serverumgebung aufgebaut werden kann. Die pfSense übernimmt dabei die zentrale Rolle als Gateway und Firewall. Durch die Kombination aus Proxmox-Bridges, internen Netzen, NAT-Regeln, UniFi-Portweiterleitung und Cloudflare-DNS-Eintrag ist ein externer Zugriff auf die internen VMs möglich, ohne die Netztrennung innerhalb der Umgebung aufzugeben.

Tailscale ergänzt die Lösung als sicherer Administrationszugang. Dadurch kann die Umgebung verwaltet werden, ohne zusätzliche Administrationsdienste direkt im Internet zu veröffentlichen.
