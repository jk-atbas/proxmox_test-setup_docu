# Einrichtung Proxmox VE 9.1 fürs lokale Testen

Installiert und konfiguriert eine Proxmox 9.1 Instanz mit graphischer Web-Oberfläche

## Vorraussetzung

- Heruntergeladene Proxmox VE 9.1 ISO [Link](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso)
- Ordner in der sich die ISO befindet. Hier unter `D:/isos`
    - Der Ordner wird auch für zukünftige ISOs dienen
- VirtualBox Installation [Link](https://www.virtualbox.org/wiki/Downloads)
- Idealerweise angeschaltener VT-d Support bei Intel Plattformen/AMD hat sowas ähnliches

## VirtualBox Setup

- Auf Schaltfläche `NEU` klicken
- VM benennen wie gewünscht
- Ordner auswählen; Hier `D:/vms`
- ISO Auswählen
- Betriebssystem kann vorausgewählt werden
    - Linux
    - Debian
    - Debian (64-bit)
- Ich würde hier keine unbeaufsichtigte Installation empfehlen, macht das initiale konfigurieren des primären Netzwerkadapters einfacher
- Hardware-Ressourcen festlegen
    - Zum Testen sollten ausreichen
        - 4 CPU
        - 4 GB Ram
        - <= 20 GB HDD
    - Kann später ggf angehoben werden
- Mit `Fertigstellen` erst einmal bestätigen. **Noch nicht VM starten**
- Rechtsklick auf die neue VM und `Ändern` auswählen
- Im neuen Fenster `Netzwerk` auswählen
    - Adapter 1 (Ist für Internetzugriff):
        - Aktiv
        - Angeschlossen an NAT
        - Virtuelles Kabel verbunden
        - Keine Port-Weiterleitungen
    - Adapter 2 (Für Zugriff von Host -> Proxmox)
        - Aktiv
        - Angeschlossen an Host-Only Adapter
        - Promiscous-Mode aus
        - Kabel verbunden
- Im neuen Fenster `Gemeinsame Ordner` auswählen
    - Neuen Gemeinsamen Ordner anlegen
        - Name egal; Hier `isos`
        - Ordnerpfad = ISOs Pfad von oben; Hier `D:/isos`
        - Einbindepunkt/Mount-Point: /mnt/shared
        - Nur lesbar
        - Automatisch einbinden
- VM Starten
- In der graphischen Übersicht des Installers den 1. Punkt `Install Proxmox VE (Graphical)` auswählen
- EULA akzeptieren
- Bei der HDD-Auswahl einfach auf `Next` klicken
- Locale ggf. auf bspw. Deutschland/Germany stellen
- Admin (Root)-Passwort vergeben
- **Achtung!** Hier muss eine valid aussehende Email eingetragen werden, die nicht der Default ist
- Netzwerk-Config:
    - Im Dropdown nic1 auswählen (Host-Only-Adapter)
    - Hostname egal
    - IP-Adresse sollte etwa so aussehen `192.168.56.x`
        - Default VirtualBox-Netz
        - Sollte logischerweise 192.168.56.2 oder größer sein
        - CIDR kann auf 24 stehen bleiben
        - Gateway auf `0.0.0.0` stellen (Später dazu mehr)
        - DNS Server auf Default stehen lassen oder sowas wie `8.8.8.8` (Google), `1.1.1.1` (Cloudfare), etc. eintragen
- Im Summary Eintragungen überprüfen
    - Könnte wie folgt aussehen ![Summary](imgs/proxmox_install_summary.png)
- Auf `Install` klicken und die Installation durchführen lassen
- Wenn die Maschine rebootet und wieder im Proxmox Install Screen landet unter `Geräte -> Optische Laufwerke -> Entfernt das virtuelle Medium aus dem Laufwerk` auswählen
- VM ausschalten/herunterfahren und dann über VirtualBox wieder starten lassen
- Im GRUB Screen `Proxmox VE GNU/Linux` auswählen und booten lassen
- Nach dem Hochfahren sollte ein Screen mit dem Text `Welcome to the Proxmox Virtual Environment. ...` stehen und darunter eine https-Adresse
    - Diese Adresse auf der Hostmaschine in einem Browser aufrufen
    - Aufruf sollte erfolgreich sein und die Web-Maske öffnen
- Login-Daten sind Nutzername `root` und das Passwort welches im Einrichtungsschritt vergeben wurde

## Proxmox-Einrichtung

- Die fehlende Lizenz-Meldung kann mit einem erneuten Neuladen der Seite ignoriert werden
- Links in der `Server View` Übersicht die Instanz unter `Datacenter` mit Rechtsklick auswählen und im Kontextmenü die `Shell` auswählen

### Internet einrichten

- Mit einem `ip link` die verfügbaren Netzwerkinterfaces prüfen
    - Die Ausgabe sollte etwa wie folgt aussehen
    ```shell
    root@pve:~ ip link
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    2: nic0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
        link/ether 08:00:27:63:eb:3b brd ff:ff:ff:ff:ff:ff
        altname enx08002763eb3b
    3: nic1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master vmbr0 state UP mode DEFAULT group default qlen 1000
        link/ether 08:00:27:3d:c2:91 brd ff:ff:ff:ff:ff:ff
        altname enx0800273dc291
    4: vmbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
        link/ether 08:00:27:3d:c2:91 brd ff:ff:ff:ff:ff:ff
    ```
    - Es sollten zwei NICs zu erkennen sein; Hier `nic0` und `nic1`
- Mit einem `nano /etc/network/interfaces` die Netzwerkadapter-Konfig öffnen
    - Den zweiten NIC/`nic1` konfigurieren:
        - Die Datei ergänzen, dass sie etwa so aussieht:
        ```shell
        auto lo
        iface lo inet loopback

        iface nic1 inet manual

        # Host-only NIC
        auto vmbr0
        iface vmbr0 inet static
                address 192.168.56.10/24
                #gateway 0.0.0.0
                bridge-ports nic1
                bridge-stp off
                bridge-fd 0

        # NAT NIC
        iface nic0 inet manual
        auto vmbr1
        iface vmbr1 inet dhcp
                bridge-ports nic0
                bridge-stp off
                bridge-fd 0
        
        source /etc/network/interfaces.d/*
        ```
        - Gateway kann hier bei `nic1` entfernt werden, da das im Host-Only-Netzwerk zwischen VM und Host nicht benötigt wird
            - Die Default-Route geht über den DHCP auf `nic0` und der NAT Verbindung
- Den Networking dienst mit `systemctl restart networking` neustarten
- Mit `ip a` alle IP-Adressen prüfen
    - Ausgabe sollte in etwa so aussehen
    ```shell
    root@pve:~ ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host noprefixroute 
        valid_lft forever preferred_lft forever
    2: nic0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master vmbr1 state UP group default qlen 1000
        link/ether 08:00:27:63:eb:3b brd ff:ff:ff:ff:ff:ff
        altname enx08002763eb3b
    3: nic1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master vmbr0 state UP group default qlen 1000
        link/ether 08:00:27:3d:c2:91 brd ff:ff:ff:ff:ff:ff
        altname enx0800273dc291
    5: vmbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
        link/ether 08:00:27:3d:c2:91 brd ff:ff:ff:ff:ff:ff
        inet 192.168.56.102/24 scope global vmbr0
        valid_lft forever preferred_lft forever
        inet6 fe80::a00:27ff:fe3d:c291/64 scope link proto kernel_ll 
        valid_lft forever preferred_lft forever
    6: vmbr1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
        link/ether 08:00:27:63:eb:3b brd ff:ff:ff:ff:ff:ff
        inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic vmbr1
        valid_lft 86389sec preferred_lft 86389sec
        inet6 fe80::a00:27ff:fe63:eb3b/64 scope link proto kernel_ll 
        valid_lft forever preferred_lft forever
    ```
- Mit `ip route` das Routing prüfen
    - Wichtig ist das die Default-Route auf sowas wie `default via 10.0.2.2 dev vmbr1` steht
- Ping-Test ins Internet machen; bspw. mit `ping -c 3 google.com`
    - Sollte erfolgreich dreimal eine Antwort erhalten

### Gemeinsamen Ordner verfügbar machen

- Im VM Fenster unter `Geräte -> Gasterweiterungen einlegen` anwählen (vorletzte Option)
- Zur Sicherheit mit einem `apt update` die apt-Package-Repos aktualisieren
    - Fehler die beim Abruf von Enterprise-Repos entstehen erstmal ignorieren
- Essentielle Build-Abhängigkeiten installieren `apt install -y build-essential dkms`
- Ordner für den Mount-Point des CD-Laufwerks vorbereiten `mkdir /mnt/cdrom`
- CD-Laufwerk mounten `mount /dev/cdrom /mnt/cdrom`
- Das VirtualBox-Installationsskript für Gästeerweiterungen ausführen `sh /mnt/cdrom/VBoxLinuxAdditions.run`
- Neustarten mit `reboot`
- Nachdem Hochfahren prüfen, ob das Kernel-Modul geladen wurde `lsmod | grep vboxsf`
    - Ausgabe sollte nicht leer sein
- Unter `/mnt/` sollte der Gemeinsame Ordner auftauchen `ls /mnt/shared`

### Gemeinsamen Ordner für Proxmox Web-GUI verwendbar machen

- Schreibbares Ziel anlegen unter `/var/lib/proxmox-iso` mit `mkdir -p /var/lib/proxmox-iso`
- Darin einen für ISO-spezifischen Unterordner anlegen `mkdir -p /var/lib/proxmox-iso/template/iso`
- Read-Only Bind-Mount für den Gemeinsamen Ordner anlegen
    - `mount --bind /mnt/shared /var/lib/proxmox-iso/template/iso`
    - `mount -o remount,ro /var/lib/proxmox-iso/template/iso`
- In der Web-GUI unter `Server View` den Knoten `Datacenter` auswählen und dann den Reiter `Storage` anklicken
    - `Add` anklicken
    - Id kann beliebig gesetzt werden
    - Als Directory `/var/lib/proxmox-iso` setzen
    - Bei Content nur `ISO Image` auswählen
    - Das Fenster mit dem Button `Add` bestätigen
- Wenn das Hinzufügen geklappt hat, dann kann man beim Anlegen einer neuen VM den Ordner anwählen
- Um das Mounting zu persistieren mit `nano /etc/fstab` öffnen und die Zeile `/mnt/shared  /var/lib/proxmox-iso/template/iso  none  bind,ro  0  0` ergänzen

### Beheben von den Package-Repo-Problemen