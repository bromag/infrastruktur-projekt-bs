# Kapitel 1: Protokollbeschreibung – Bestellung und Zugriff auf eine VM

## 1.1 Ziel dieses Kapitels

Dieses Kapitel beschreibt einfach und verständlich, wie eine virtuelle Maschine bestellt und anschliessend verwendet werden kann. Die Beschreibung ist aus Kundensicht geschrieben. Der Kunde muss keine technischen Details zu Proxmox, pfSense oder Netzwerken kennen.

Im Mittelpunkt stehen folgende Fragen:

- Wie kann eine VM bestellt werden?
- Welche Informationen muss der Kunde liefern?
- Was richtet Basel-GmbH ein?
- Welche Zugangsdaten erhält der Kunde?
- Wie verbindet sich der Kunde mit der VM?

## 1.2 Wie bestellt der Kunde eine VM?

Der Kunde kontaktiert die Basel-GmbH und teilt mit, dass er eine VM benötigt. Die Anfrage kann zum Beispiel per E-Mail erfolgen.

Beispiel:

```text
Guten Tag

Ich benötige eine virtuelle Maschine für Testzwecke.
Die VM soll von extern per SSH erreichbar sein.

Freundliche Grüsse
<Kunde>
```

## 1.3 Welche Informationen muss der Kunde angeben?

Damit die VM korrekt erstellt werden kann, muss der Kunde einige Angaben machen.

\begin{center}
\renewcommand{\arraystretch}{1.2}
\begin{tabular}{|p{4cm}|p{6cm}|p{4cm}|}
\hline
\textbf{Angabe} & \textbf{Erklärung} & \textbf{Beispiel} \\
\hline
Zweck der VM & Wofür wird die VM verwendet? & Testsystem \\
\hline
Betriebssystem & Welches System soll installiert werden? & Ubuntu Server \\
\hline
Benutzername & Mit welchem Namen möchte sich der Kunde anmelden? & \texttt{kunde01} \\
\hline
Zugriff & Wie möchte der Kunde auf die VM zugreifen? & SSH \\
\hline
SSH Public Key & Schlüssel für den sicheren Zugriff & \texttt{ssh-ed25519 …} \\
\hline
Laufzeit & Wie lange wird die VM benötigt? & bis Projektende \\
\hline
\end{tabular}
\end{center}

Der wichtigste Punkt ist der **SSH Public Key**. Damit kann sich der Kunde sicher auf der VM anmelden, ohne dass ein Passwort per E-Mail verschickt werden muss.

## 1.4 Was macht Basel-GmbH?

Nachdem die Bestellung eingegangen ist, erstellt die Basel-GmbH die VM auf der Proxmox-Umgebung.

Der Basel-GmbH erledigt dabei folgende Schritte:

1. Sie prüft, ob genügend Ressourcen vorhanden sind.
2. Sie erstellt die VM.
3. Sie installiert das gewünschte Betriebssystem.
4. Sie erstellt den Benutzer für den Kunden.
5. Sie hinterlegt den SSH Public Key des Kunden.
6. Sie richtet den externen Zugriff ein.
7. Sie testet, ob die VM erreichbar ist.
8. Sie sendet dem Kunden die Zugriffsinformationen.

## 1.5 Welche Informationen erhält der Kunde?

Nach der Bereitstellung erhält der Kunde eine kurze Rückmeldung mit den wichtigsten Angaben.

Beispiel:

```text
Guten Tag

Ihre virtuelle Maschine wurde erstellt.

VM-Name: vm-kunde01
Betriebssystem: Ubuntu Server
DNS-Name: pfsense-bs.athena-forge.ch
SSH-Port: 2222
Benutzername: kunde01

Sie können sich mit folgendem Befehl verbinden:
ssh kunde01@pfsense-bs.athena-forge.ch -p 2222

Freundliche Grüsse
<Basel-GmbH>
```

## 1.6 Übersicht des Kundenzugriffs

![Kundensicht VM-Zugriff](images/kundensicht_vm_zugriff_einfach.drawio.png){ width=100% }

Die Darstellung zeigt den Zugriff aus Sicht des Kunden. Der Kunde muss keine technischen Details zur Infrastruktur kennen. Für ihn sind nur die erhaltenen Zugangsdaten wichtig.

Der Kunde erhält von Basel-GmbH einen Benutzernamen, einen DNS-Namen, einen Port und die Information, welcher SSH-Schlüssel verwendet wird. Mit diesen Angaben kann er sich von seinem eigenen Computer aus mit der VM verbinden.

Der DNS-Name, zum Beispiel `pfsense-bs.athena-forge.ch`, dient als Adresse zur Umgebung von Basel-GmbH. Der Port entscheidet, welche VM erreicht wird. Wenn mehrere VMs über dieselbe Adresse erreichbar sind, wird jede VM über einen eigenen Port angesprochen.

Beispiel:

\begin{center}
\renewcommand{\arraystretch}{1.2}
\begin{tabular}{|p{7cm}|p{3cm}|p{3cm}|}
\hline
\textbf{DNS-Name} & \textbf{Port} & \textbf{Ziel} \\
\hline
\texttt{pfsense-bs.athena-forge.ch} & \texttt{2222} & VM 1 \\
\hline
\texttt{pfsense-bs.athena-forge.ch} & \texttt{2223} & VM 2 \\
\hline
\end{tabular}
\end{center}

Der Kunde muss sich nicht darum kümmern, wie die Verbindung intern weitergeleitet wird. Diese Weiterleitung wird von Basel-GmbH eingerichtet und betrieben. Für den Kunden reicht es aus, den bereitgestellten SSH-Befehl zu verwenden.

Beispiel:

```bash
ssh kunde01@pfsense-bs.athena-forge.ch -p 2222
```

## 1.7 Warum braucht es einen Port?

Da mehrere VMs über dieselbe öffentliche Adresse erreichbar sein können, erhält jede VM einen eigenen Port. Der Port ist vergleichbar mit einer Türnummer. Die Adresse zeigt zum richtigen Standort, der Port zeigt zur richtigen VM.

Beispiel:

\begin{center}
\renewcommand{\arraystretch}{1.2}
\begin{tabular}{|p{7cm}|p{3cm}|p{3cm}|}
\hline
\textbf{Adresse} & \textbf{Port} & \textbf{Ziel} \\
\hline
\texttt{pfsense-bs.athena-forge.ch} & \texttt{2222} & VM 1 \\
\hline
\texttt{pfsense-bs.athena-forge.ch} & \texttt{2223} & VM 2 \\
\hline
\end{tabular}
\end{center}

Der Kunde muss sich nur den DNS-Namen, den Benutzernamen und den Port merken.

## 1.8 Einfacher Ablauf zusammengefasst

Der gesamte Ablauf sieht vereinfacht so aus:

```text
1. Kunde bestellt eine VM
2. Kunde gibt Zweck, Benutzername und SSH Public Key an
3. Basel-GmbH erstellt die VM
4. Basel-GmbH richtet den Zugriff ein
5. Kunde erhält DNS-Name, Benutzername und Port
6. Kunde verbindet sich per SSH mit der VM
```