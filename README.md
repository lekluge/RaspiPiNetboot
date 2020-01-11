
# RaspiPiNetboot



## Beschreibung

Der Netzwerkboot ist schon seit Jahren fester bestandteil eines Modernen Computers. Nun wird dieser auch unter den Raspberry Pi's möglich. Das bietet eine Große Reihe an Vorteilen: 
	- Kein R/W mehr auf der SD Karte
	- mehr Speicherplatz
	- Möglichkeit OS schnell zu wechseln
	- Verschiedene OS auf Verschiedenen Pi`s 
	- Ein System alle Pi`s von einem Punkt zu steuern

## Vorbereitung

Das einzige was ihr benötigt ist min. 1 Raspberry Pi (Version egal) mit einer SD Karte min (8GB), einen Server oder eine VM mit einem Debian System und ein paar LAN-Kabel. Gegebenenfalls würde ich euch noch einen Networkswitch empfehlen.

## Installation
Jetzt gehts an die Installation. Diese ist nicht ganz simple und ein einfacher Fehler kann zu großer Frustration führen (glaubt mir..). Ich rate euch daher meiner Anleitung Schritt für Schritt zu Folgen.

### Server Vorbereitung

Als erstes kümmern wir uns um den Server da das mit das einfachste ist. Wie oben bereits erwähnt brauchen wir ein Debian System. Ich rate hier zum ganz simplen Debian was ihr [hier](https://www.debian.org/ "hier") finden könnt.
Um nicht unnötig Leistung zu verschwenden Rate ich euch bei der Installation auf das **Software Auswahl** Fenster zu achten. Wählt dort nur den **SSH Server** und die **Standard System Anwendungen**. Wenn ihr wollt könnt ich natürlich eine Grafische Oberfläche installieren aber wie gesagt zieht das unnötig Leistungen.

Sobald das Debian System installiert ist meldet euch an und updatet euer System auf den aktuellen Stand mit folgendem Befehl: 

```
sudo apt-get update && sudo apt-get upgrade
```

Danach werden die notwendigen Komponenten installiert:

```
sudo apt-get install dnsmasq rpcbind nfs-kernel-server
```
Nachdem alle Pakete installiert sind müssen wir noch ein paar Ordner erstellen. Zum einen den Ordner für die Bootdateien:
```
sudo mkdir /tftpboot
sudo chmod -R 777 /tftpboot
```
Und den Ordner für das RootFileSystem(RFS) 
```
sudo mkdir /nfs/client1 #Kann beliebig umbenannt werden
``` 
Als nächstes brauch der Server selber eine Statische IP Adresse. Dafür brauchen wir erst einmal den Namen des Netzwerk Adapters: 
```
ifconfig
```
Meistens wird der Server über LAN angeschlossen und wird daher die Bezeichnung eth0 haben. Es kann jedoch auch andere Namen geben. 

Danach legen wir die statische IP fest in dem wir die interfaces Datei modifizieren. 
```
sudo nano  /etc/network/interfaces
```
Falls dort eine Zeile wie `iface eth0 inet manual` steht dann kommentiert diese einfach aus und fügt dann Folgende Konfiguration ein
```
port=0 # schaltet den DHCP Server ab
dhcp-range=10.42.1.255,proxy #Die IP Range die genutzt werden soll
log-dhcp # Der DHCP Server(eigl. nur TFTP) schreibt log Daten
enable-tftp #TFTP Server wird eingeschaltet
tftp-root=/tftpboot #Path zum Boot Ordner
pxe-service=0,"Raspberry Pi Boot" #Name des Service(NICHT ÄNDERN!)
tftp-unique-root #Möglichkeit mehrer Boot Ordner
```
Danach stellen müssen wir den Service für einen Neustart aktivieren und Neustarten
```
sudo systemctl enable dnsmasq
sudo systemctl restart dnsmasq
```
Jetzt stellen wir noch den Service für das RFS ein. Dazu bearbeiten wir die Datei `/etc/exports` und fügen Folgende Einträge hinzu:
```
/nfs/client1 *(rw,sync,no_subtree_check,no_root_squash)
```
Je nachdem ob ihr den Ordner anders benannt habt müsst ihr den Namen ebenfalls ändern. Und wenn ihr ein weiteres RFS hinzufügt braucht ihr ebenfalls einen neuen Ordner und den dazu gehörigen Eintrag.
Danach müssen wir hier ebenfalls die Services aktivieren und  Neustarten:
```
sudo systemctl enable rpcbind
sudo systemctl enable nfs-kernel-server
sudo systemctl restart rpcbind
sudo systemctl restart nfs-kernel-server
```
Nach dem wir das gemacht haben sind wir an dem Server so gut wie fertig es fehlen nur noch die Bootfiles und das RFS das installiert werden soll. Darauf kommen wir allerdings später zurück. Den wir müssen uns jetzt erst mal um den Client also um den Pi kümmern.

### Client Vorbereitung 
Jetzt müssen wir den Raspberry Pi einrichten was leicht klingt aber trotzdem ein paar Schwierigkeiten mit sich bringt. Zur nächst sollten wir uns überlegen welches System wir später auf den Pi laufen lassen wollen und auch hierbei gibt es Unterschiede.

####  Rasbian Lite/Full

#### Kodi

 




