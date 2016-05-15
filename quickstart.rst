Server Installation
===================

In den folgenden Schritten wird die Installation von Tunneldigger für Gluon beschrieben.
Dieses Beispiel nutzt Debian Jessie, für andere Distributionen müssen die Befehle angepasst werden.

Source Code
-----------

Die Tunneldigger-Sources können mit dem folgenden Befehl heruntergeladen werden::

    git clone git://github.com/ffrl/tunneldigger.git

Dies erstellt einen Ordner mit dem Namen ``tunneldigger``. In diesem befindet sich der Quelltext für den Client und den Broker.
Dieser Scritt ist nicht Teil der Installation und dient lediglich der Vollständigkeit dieser Dokumentation.

Abhängigkeiten
--------------

Tunneldigger benötigt einen Linux Kernel (>= 2.6.36), welcher L2TPv3 unterstützt.
Für den Broker-Dienst müssen die folgenden Kernel-Module geladen sein:

* l2tp_core
* l2tp_eth
* l2tp_netlink
* ebtables

Zusätzlich muss der Kernel NAT via Netfilter unterstützen, ansonsten kann Tunneldigger nicht alle Tunnel über den selben externen Port laufen lassen. Dies ist normalerweise der Fall.
Diese Modules sollten beim booten geladen werden, indem diese in ``/etc/modules`` hinterlegt werden.

Zu den Kernel Modulen werden auch noch Software Pakete benötigt. Auf Debian sind es folgende::

* iproute
* bridge-utils
* libnetfilter-conntrack3
* python-dev
* libevent-dev
* ebtables
* python-virtualenv

Tunneldigger wird in einem so genanten "Python Virtual Environment" betrieben, dadurch können die Python-Pakete für Tunneldigger 
seperat von den Python Paketen des Betriebssystems installiert werden. Dadurch soll vermieden werden, das die Pakete sich gegenseitig stören.

Um die oben aufgeführten Pakete zu installieren kann der folgende Befehl ausgeführt werden::

    sudo apt-get install iproute bridge-utils libnetfilter-conntrack3 python-dev libevent-dev ebtables python-virtualenv 

Installation
------------

Diese Anleitung installiert den Broker unter ``/srv/tunneldigger``.
Die Installation geschieht mittels:

    mkdir /srv
    cd /srv
    git clone git://github.com/ffrl/tunneldigger.git
    virtualenv tunneldigger

Als nächster Schritt werden die Abhängigkeiten für den Broker installiert.
Dafür aktivieren wir die virtuelle Python-Umgebung und installieren die Pakete aus
der Datei ``requirements.txt``

    cd /srv/tunneldigger
    . bin/activate
    pip install -r broker/requirements.txt

Konfiguration
-------------

Der Broker benötigt eine Konfigurations-Datei als Commandline-Option, ein Beispiel befindet sich in ``broker/l2tp_broker.cfg.example``. Diese Datei kopieren wir nach ``/srv/tunneldigger/l2tp_broker.cfg`` und passen diese an.
Ein paar Optionen müssen angepasst werden, der Rest kann in der Regel belassen werden.

* **address** hier muss die externe IP-Adresse des Brokers eingetragen werden, zu welcher die Clients sich verbinden.

* **port** hier werden die Ports eingetragen, auf welchen der Broker erreichbar ist. Es können mehrere Ports eingetragen werden (Durch Komma getrennt).

* **interface** der Name des Interfaces auf dem der Broker lauscht.

* **pmtu-discovery** muss auf "false" gesetzt werden, da Batman-Advanced keine dynamische MTU-Änderung unterstützt.

* Die Hooks im Abschnitt **hooks** müssen mit den Pfaden der Interface-Scripts konfiguriert werden. Wenn dies nicht der Fall ist dann werden Verbindungen nur aufgebaut, aber die Interfaces nicht konfiguriert.


Hooks
-----

Momentan existieren vier verschiedene hooks.
* ``session.up`` wird aufgerufen nachdem das Interface vom Broker erstellt wurde und bereit für die weitere Konfiguration ist. (Hier muss ``/srv/tunneldigger/scripts/session-up.sh`` eingetragen werden)

* ``session.pre-down`` wird kurz vor dem Entfernen des Interfaces durch den Broker aufgerufen. (Hier muss ``/srv/tunneldigger/scripts/session-pre-down.sh`` eingetragen werden)

* ``session.down`` wird aufgerufen nachdem das Interface entfernt wurde und nicht mehr nutzbar ist. (Dies wird momentan nicht genutzt)

* ``session.mtu-changed`` wird bei MTU-Änderungen aufgerufen (Dieser Hook wird momentan nicht genutzt, da die Clients wegen Batman-Advanced keine dynamischen MTU-Änderungen unterstützen.)

Hook-Scripts
------------

Die folgenden Hook-Scripts müssen unter ``/srv/tunneldigger/scripts`` abgelegt werden.

Script: ``/srv/tunneldigger/scripts/session-up.sh``
```
#!/bin/bash
INTERFACE="$3"
UUID="$8"

log_message() {
    message="$1"
    logger -p 6 -t "Tunneldigger" "$message"
    echo "$message" | systemd-cat -p info -t "Tunneldigger"
    echo "$1" 1>&2
}

if /bin/grep -Fq $UUID /srv/tunneldigger/blacklist.txt; then
        log_message "New client with UUID=$UUID is blacklisted, not adding to tunneldigger bridge interface"
else
        log_message "New client with UUID=$UUID connected, adding to tunneldigger bridge interface"
        ip link set dev $INTERFACE up mtu 1364
        /sbin/brctl addif tunneldigger $INTERFACE
fi
```

Script: ``/srv/tunneldigger/scripts/session-pre-down.sh``
```
#!/bin/bash
INTERFACE="$3"

/sbin/brctl delif tunneldigger $INTERFACE
exit 0
```

Nicht vergessen die Scripts mittels ``chmod +x`` ausführbar zu machen!

Client-Blacklist
----------------

Wie bei Fastd können Clients ausgesperrt werden, dafür wird die Datei ``/srv/tunneldigger/blacklist.txt`` genutzt.
Hier können die NodeIDs der zu sperrenden Clients eingetragen werden, jeweils in einer Zeile.

Betriebssystem-Konfiguration
============================

Nach der Konfiguration von Tunneldigger müssen noch ein paar Dinge im Betriebssystem angelegt werden, ein Start-Script, eine Systemd-Unit, sowie die Bridge Konfiguration.

Start-Script und Systemd Unit
-----------------------------

Script: ``/srv/tunneldigger/start-broker.sh``
```
#!/bin/bash

WDIR=/srv/tunneldigger
VIRTUALENV_DIR=/srv/tunneldigger

cd $WDIR
source $VIRTUALENV_DIR/bin/activate

bin/python broker/l2tp_broker.py l2tp_broker.cfg
```

Script: ``/etc/systemd/system/tunneldigger.service``
```
[Unit]
Description = Start tunneldigger L2TPv3 broker
After = network.target

[Service]
ExecStart = /srv/tunneldigger/start-broker.sh

[Install]
WantedBy = multi-user.target
```

Anschließend aktivieren wir den Tunneldigger Dienst, damit dieser beim booten startet::

    systemctl enable tunneldigger.service

Tunneldigger-Bridge
-------------------

Anschließend wird die Tunneldigger Bridge konfiguriert. Alle Tunnel werden in einer Bridge zusammengefasst da Batman-Advanced nicht mit vielen Interfaces umgehen kann.
Damit dies nicht zu Problemen mit Batman führt, müssen die Clients untereinander isoliert werden, denn die Kommunikation zwischen den Clients übernimmt Batman-Advanced.
Dazu legen wir die folgende Datei an:

Datei: ``/etc/network/interfaces.d/tunneldigger``
```
# Tunneldigger VPN Interface
auto tunneldigger
iface tunneldigger inet manual
  ## Bring up interface
  pre-up brctl addbr $IFACE
  pre-up ip link set address 0A:BE:EF:25:00:01 dev $IFACE
  pre-up ip link set dev $IFACE mtu 1364
  pre-up ip link set $IFACE promisc on
  up ip link set dev $IFACE up
  post-up ebtables -A FORWARD --logical-in $IFACE -j DROP
  post-up batctl if add $IFACE
  # Shutdown interface
  pre-down batctl if del $IFACE
  pre-down ebtables -D FORWARD --logical-in $IFACE -j DROP
  down ip link set dev $IFACE down
  post-down brctl delbr $IFACE
```

Hierbei muss die Interface MTU nach eigenen Wünschen angepasst werden, wir nutzen hier 1364 welches in Tests die besten Ergebnisse lieferte.
Außerdem sollte eine eindeutige MAC Adresse für jeden Broker gewählt werden.


Zum Abschluss starten wir das Tunneldigger-Bridge Interface sowie den Broker

    ifup tunneldigger
    systemctl start tunneldigger
