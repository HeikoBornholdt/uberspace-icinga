# Icinga auf einem Uberspace betreiben

Diese Anleitung beschreibt die Installation von Icinga innerhalb eines [Uberspaces](https://uberspace.de/).

## Installation

Zuerst wird der Quelltext auf dem Uberspace herunterladen und anschließend entpackt:

    mkdir ~/src
    cd ~/src/
    wget https://github.com/Icinga/icinga-core/releases/download/v1.10.3/icinga-1.10.3.tar.gz
    tar xzf icinga-1.10.3.tar.gz

    cd icinga-1.10.3/
    ./configure --prefix=/home/$USER/opt/icinga \
    --with-icinga-user=$USER \
    --with-icinga-group=$USER \
    --with-command-user=$USER \
    --with-command-group=$USER \
    --with-web-user=$USER \
    --with-web-group=$USER \
    --with-httpd-conf=/home/$USER/opt/icinga/etc/apache2 \
    --with-init-dir=/home/$USER/opt/icinga/etc/init.d \
    --with-cgiurl=/cgi-bin/icinga \
    --disable-idoutils

Trotz des angegebenen Benutzers/der angegebenen Gruppe müssen wir noch händisch die Datei `Makefile` anpassen.

In Zeile 58, bei `INIT_OPTS` muss `root` gegen den eigenen Benutzer-/Gruppennamen ausgetauscht werden.

    sed -i "/^INIT_OPTS=/s/root/$USER/g" Makefile

Jetzt müssen noch ein paar Verzeichnisse/Symlinks erstellt werden, damit die Dateien direkt in den vorgesehenen Verzeichnissen landen:

    mkdir -p ~/opt/icinga/var/{rw,lock}
    mkdir -p ~/opt/icinga/etc/{apache2,init.d}
    mkdir /var/www/virtual/$USER/html/icinga
    mkdir /var/www/virtual/$USER/cgi-bin/icinga
    ln -s /var/www/virtual/$USER/cgi-bin/icinga ~/opt/icinga/sbin
    ln -s /var/www/virtual/$USER/html/icinga ~/opt/icinga/share

Jetzt können wir Icinga kompilieren:

    make all
    make fullinstall
    
Im Falle einer Neuinstallation muss die Konfigration erzeugt werden:

    make install-config

Damit suEXEC die Ausführung der .cgi-Dateien nicht verweigert, müssen deren Zugriffsrechte angepasst werden:

    chmod -R 755 /var/www/virtual/$USER/cgi-bin/icinga

Nun muss die Datei `/home/$USER/opt/icinga/etc/icinga.cfg` angepasst werden:

`use_syslog` ist dabei auf `0` zu setzen

    sed -i '/use_syslog=1/s/1/0/' /home/$USER/opt/icinga/etc/icinga.cfg

`/home/$USER/opt/icinga/etc/init.d/icinga` benötigt ebenfalls eine Anpassung: 

`IcingaLockDir` ist dabei auf `${prefix}/var/lock/subsys` zu setzen.

    sed -i '/^IcingaLockDir=/s/=/=${prefix}/' /home/$USER/opt/icinga/etc/init.d/icinga

Nun wird die Weboberfläche von Icinga noch gegen unbefugte Zugriffe abgesichert:

    htpasswd -s -c /var/www/virtual/$USER/html/icinga/.htpasswd icingaadmin

    echo 'AuthName "Icinga Access"
    AuthType Basic
    AuthUserFile /var/www/virtual/'$USER'/html/icinga/.htpasswd
    Require valid-user' > /var/www/virtual/$USER/html/icinga/.htaccess

Die CGI-Skripte von Icinga sollten auch geschützt werden:

    cp /var/www/virtual/$USER/html/icinga/.htaccess /var/www/virtual/$USER/cgi-bin/icinga/

Jetzt noch Icinga ausführbar machen:

    chmod +x ~/opt/icinga/etc/init.d/icinga 

Icinga selber ist nun fertig installiert und kann auch schon gestartet werden:

  * Icinga-Konfiguration prüfen: `~/opt/icinga/etc/init.d/icinga checkconfig`
  * Icinga starten: `~/opt/icinga/etc/init.d/icinga start`

Damit Icinga beim Neustart des Servers ebenfalls mitgestartet wird, muss ein [daemon
eingerichtet werden](https://wiki.uberspace.de/system:daemontools#einen_daemon_einrichten).
Das run-Skript kann dabei wie folgt aussehen:

    #!/bin/sh

    # These environment variables are sometimes needed by the running daemons
    export USER=DEIN_USERNAME
    export HOME=/home/DEIN_USERNAME

    # Include the user-specific profile
    . $HOME/.bash_profile

    # Now let's go!
    exec ~/opt/icinga/bin/icinga ~/opt/icinga/etc/icinga.cfg 2>&1

### Nagios Plugins

Die ganzen Skripte mit den Prüfroutinen (`check_xyz`) müssen separat installiert werden:

    cd ~/src/
    
    wget wget https://www.nagios-plugins.org/download/nagios-plugins-2.0.tar.gz
    tar xzf nagios-plugins-2.0.tar.gz

    cd nagios-plugins-2.0/
    ./configure --prefix=/home/$USER/opt/icinga \
    --with-nagios-user=$USER \
    --with-nagios-group=$USER 

    make
    make install

### NRPE

Wer andere Server via NRPE überwachen möchte benötigt `check_nrpe`:

    cd ~/src/
    wget http://sourceforge.net/projects/nagios/files/nrpe-2.x/nrpe-2.15/nrpe-2.15.tar.gz/download -O nrpe-2.15.tar.gz
    tar xzf nrpe-2.15.tar.gz

    cd nrpe-2.15/
    ./configure --prefix=/home/$USER/opt/icinga \
    --with-nagios-user=$USER \
    --with-nagios-group=$USER \
    --with-nrpe-user=$USER \
    --with-nrpe-group=$USER

    make
    make install
