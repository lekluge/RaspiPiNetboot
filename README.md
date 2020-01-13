
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
sudo mkdir /nfs/
``` 
Als nächstes braucht der Server selber eine Statische IP Adresse. Dafür brauchen wir erst einmal den Namen des Netzwerk Adapters: 
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
auto eth0
allow hotplug
iface eth0 inet static
address 10.41.1.21 #Die Static Ip die ihr dem Server geben wollt
netmask 255.255.255.0 #Die Netmask könnt ihr im Normalfall genauso übernehmen
gateway 10.41.1.1 #Ist im Normalfall immer eure IP und als Endziffer die 1
dns-server 10.41.1.1 # Ist im Normalfall immer gleich mit dem Gateway
```

Außerdem müssen wir DNSmasq noch sagen das er den TFTP Server bereitstellen soll. Editiert dafür die Datei /etc/dnsmasq/dnsmasq.conf und fügt diese Konfiguration ein.
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
/nfs/flurpi *(rw,sync,no_subtree_check,no_root_squash)
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

####  Raspbian Lite/Full

Um ein Raspbian Lite oder Full Image zu erstellen das auch für den Netzwerk Boot geiegnet ist müssen wir dieses ersteinmal ganz normal erstellen. Sprich wir Laden uns das Image von der offizielen Raspberry Pi Seite herunter installieren es wie es in der Dokumentation beschrieben ist.

Nach dem ihr auf dem Pi ein vollständiges Raspbian System am laufen habt können wir Anfangen das System vorzubereiten. Als erstes werden wir das System Updaten damit es später auch auf dem neusten Stand ist.
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
```
Nach dem das Abgeschlossen ist müssen wir das Swap System deaktivieren und entfernen, das geht recht simple mit folgenden Befehlen. 
```
sudo dphys-swapfile swapoff
sudo dphys-swapfile uninstall
sudo update-rc.d dphys-swapfile remove
```
Jetzt müssen wir noch ein Komplettes Backup von dem RFS machen. Damit wir dieses gleich nicht unnötig suchen müssen erstellen wir einen Ordner, hierbei ist es Sinnvoll einen Namen zu wähle der für den späteren Einsatz Sinnvoll ist, sprich wollt ihr den Pi im Flur aufstellen würde ich den Namen 'FlurPi' empfehlen.
```
mkdir -p /nfs/flurpi
rsync -xa --progress --exclude /nfs / /nfs/flurpi
```
Das kann eine Weile dauern, bei mir hat das im Schnitt ca 20 min gedauert. Nach dem dieser Vorgang abgeschlossen ist müssen wir noch die SSH Keys neu generieren. Um das zu tun mounten wir das System und loggen uns über ``chroot`` ein.
```
cd /nfs/flurpi
sudo mount --bind /dev dev
sudo mount --bind /sys sys
sudo mount --bind /proc proc
rm /etc/ssh/ssh_host_*
dpkg-reconfigure openssh-server
exit
sudo umount dev sys proc
```
Nun müssen wir gegebenenfalls noch die Swap Datei löschen falls sie denn noch Vorhanden ist. 
```
sudo rm /nfs/client1/var/swap
```

Damit sind wir mit dem Root File System auch schon Fertig nun müssen wir dieses nur noch auf den Server übertragen. Hierbei gilt es zu beachten das Filesystem nicht einfach per FTP zu übertragen da sonst die Rechte korruptiert werden könnten. Daher packen wir das ganze Filesystem in einer tar Datei.

```
tar -cpf /flurpi.tar /nfs
```

Sobald der Vorgang abgeschlossen ist können wir die tar Datei entweder per FTP übertragen oder man nimmt die SD Karte aus dem Pi und kopiert die Datei dann direkt auf den Server. Auch beim entpacken gibt es etwas zu beachten. Geht dazu zu dem Abschnitt "**Server und Client abschließen**".

#### Kodi ohne apt

**COMING SOON** 

#### Kodi mit apt

Kodi mit der Möglichkeit Packages über apt zu installieren gibt es theoretisch schon und zwar mit OSMC. Allerdings ist OSMC nicht netboot fähig weshalb wir einen etwas umständlicheren Weg gehen müssen. 
Als erstes folg ihr die ganz normale Installation von Raspbian aber um Ressourcen zu sparen nutzt bitte Raspbian Lite.

Nach dem die Installation abgeschlossen ist und ihr euch ganz normal ins Raspbian Lite über Netboot einloggen könnt, können wir das Kodi Package ganz einfach nach installieren. 
```
sudo apt-get update
sudo apt-get install kodi
```

Danach lässt sich Kodi ganz einfach startet. Wenn ihr Kodi jetzt noch starten wollt wenn der Pi hochfährt müsst ihr diesen nur noch in den Autostart packen. 
```
sudo crontab -e
```
Dort fügt ihr dann folgenden Autostart Befehl ein
```
@reboot kodi --standalone
```
 
### Server und Client abschließen

#### Server abschließen

Wenn ihr hier angekommen seit sind wir auch schon fast fertig. Jetzt geht es nur noch darum das RFS zu entpacken und die Bootdatein zur Verfügung zu stellen. Außerdem müssen wir den Pi noch mit der Boot datei ausrüsten. Als erstes kümmern wir uns darum das RFS auf den Server zu übertragen und dann ordentlich zu entpacken. Steckt dafür die SD Karte per USB an den Server oder übertragt das ganze per FTP. 
Wenn ihr euch für USB entschieden habt dann folgt diesen Schritten
```
lsblk
```
Um zu schauen welche Partition die Richtige ist. In meinem Fall ist das /dev/sdb1. Also mounten wir erstmal diese Partition. Ich rate hierfür ein eigenes Verzeichnis anzulegen
```
sudo mkdir /mnt/sd1
sudo mount /dev/sdb1 /mnt/sd1
```
Danach kopieren wir die vorher erstellte tar Datei auf den Server und entpacken diese dann direkt.
```
sudo cp /mnt/sd1/nfs.tar /
sudo tar --same-owner -xvf nfs.tar
```
Jetzt solltet ihr in eurem NFS Ordner auf dem Server das RFS von dem Pi haben in einem Ordner Namens "flurpi". Soweit so gut jetzt brauchen wir noch die Bootdatein die der Server an den Pi senden soll. 
Diese könnt ihr euch einfach aus meiner Repo runterladen und dann in den tftpboot Orden packen.

**Wenn ihr mehrere Pi's per Netboot starten wollt, überspringt diesen Schritt und geht zur "Mehre Pi's" Sektion** 

```
git clone https://git.chaospott.de/t1ggy/RaspiNetBoot.git
cp /RaspiNetBoot/bootfiles/* /tftpboot
```
Danach müssen wir nur noch die ``cmdline.txt`` etwas editieren. Diese sieht wie folgt aus.
```
console=serial0,115200 console=tty1 root=/dev/nfs nfsroot=10.42.1.21:/nfs/flurpi rw ip=dhcp rootwait elevator=deadline
```
Hier müsst ihr eigentlich nur eure Static IP von dem Server eintragen die ihr vorher Festgelegt habt und den Namen eures Ordners in dem NFS Verzeichnis.
Jetzt müssen wir nur noch die fstab Datei ändern da der Pi sonst nach einem Filesystem auf der SD Karte sucht. Löscht dafür alle Eintrage bis auf **proc** aus der Datei
```
sudo nano /nfs/flurpi/etc/fstab
```
Danach sollte die Datei so aussehen
```
proc  /proc  proc  defaults  0  0
```

Jetzt ist der Server vollständig konfiguriert Und Einsatz bereit. Um den Status des Netbootes zu Verfolgen könnt ihr die log Datei auslesen. 
```
tail /var/log/daemon.log -f
```

#### Client abschließen

Das was wir jetzt noch machen müssen damit der Pi endlich über das Netzwerk gebootet werden kann ist die Bootdatei auf die SD Karte zu packen. Dazu sollten wir die SD Karte aber ersteinmal Partitionieren. Um zu sehen welche bezeichung eure sd Karte hat gebt 
```
lsblk
```
ein. In meinem Fall ist das /dev/sde. Also schnell fdisk öffnen und die vorhandenen Partitionen löschen und eine neue 50MB große Partition anlegen. 
```
sudo fdisk /dev/sdb
```
Um das nun so einfach wie möglich zu erklären zeige ich euch einfach meine Ausgabe. 
```
Welcome to fdisk (util-linux 2.30.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): p
Disk /dev/sde: 1.9 GiB, 2002780160 bytes, 3911680 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Device     Boot Start     End Sectors  Size Id Type
/dev/sde1        2048 3911679 3909632  1.9G  b W95 FAT32

Command (m for help): d
Selected partition 1
Partition 1 has been deleted.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): 

Using default response p.
Partition number (1-4, default 1): 
First sector (2048-3911679, default 2048): 
Last sector, +sectors or +size{K,M,G,T,P} (2048-3911679, default 3911679): +50M

Created a new partition 1 of type 'Linux' and of size 50 MiB.
Partition #1 contains a vfat signature.

Do you want to remove the signature? [Y]es/[N]o: Y

The signature will be removed by a write command.

Command (m for help): t
Selected partition 1
Hex code (type L to list all codes): b 
Changed type of partition 'Linux' to 'W95 FAT32'.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Synching disks.
```

Wenn ihr nun eure Partition erstellt habt könnt ihr fdsik einfach mit Strg + C beenden.
Nun müssen wir unsere eben erstellte Partition nur noch Formatieren.
```
sudo mkfs.fat -n boot /dev/sde1
```

Jetzt mounten wir genau diese Partition wieder und übertragen unsere Bootdatei
```
sudo mount /dev/sde1 /mnt/sd1
cp /tftpboot/bootcode.bin /mnt/sd1/
```

Und das wars auch schon jetzt einfach den Pi per Lan anschließen die SD Karte einstecken und neu starten. 
Danach sollte auf eurem Server(sofern ihr die Log Datei auslest) eine Meldung auftauchen das der Pi Daten anfragt und diese übertragen werden. Nach einer Kurzen Zeit sollte der Pi dann starten und ihr könnt ihn ganz Normal verwenden. 


