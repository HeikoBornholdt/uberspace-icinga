# Icinga auf einem Uberspace betreiben

Diese Anleitung basiert auf der Anleitung "Nagios ohne root-Rechte auf dem Uberspace" von Tom Förster (https://tom-f.org/blog/nagios-ohne-root-rechte-auf-dem-uberspace/) und wurde lediglich von mir an Icinga angepasst.

## Installation

Zuerst wird der Quelltext auf dem Uberspace herunterladen und anschließend entpackt:

    mkdir ~/src
    cd ~/src/
    wget http://sourceforge.net/projects/icinga/files/icinga/1.8.4/icinga-1.8.4.tar.gz
    tar xzf icinga-1.8.4.tar.gz

    cd icinga-1.8.4/
    ./configure --prefix=/home/$USER/opt/icinga \
    --with-icinga-user=$USER \
    --with-icinga-group=$USER \
    --with-command-user=$USER \
    --with-command-group=$USER \
    --with-web-user=$USER \
    --with-web-group=$USER \
    --with-httpd-conf=/home/$USER/opt/icinga/etc/apache2 \
    --with-init-dir=/home/$USER/opt/icinga/etc/init.d \
    --with-cgiurl=/cgi-bin/icinga

Trotz des angegebenen Benutzers/der angegebenen Gruppe müssen wir noch händisch die Datei `Makefile` anpassen.

In Zeile 48, bei `INSTALL_OPTS` muss `root` gegen den eigenen Benutzer-/Gruppennamen ausgetauscht werden.

Jetzt müssen noch ein paar Verzeichnisse/Symlinks erstellt werden, damit die Dateien direkt in den vorgesehenen Verzeichnissen landen:

    mkdir -p ~/opt/icinga
    mkdir -p ~/opt/icinga/var/rw
    mkdir -p ~/opt/icinga/var/lock
    mkdir -p ~/opt/icinga/etc/apache2
    mkdir -p ~/opt/icinga/etc/init.d
    mkdir /var/www/virtual/$USER/html/icinga
    mkdir /var/www/virtual/$USER/cgi-bin/icinga
    ln -s /var/www/virtual/$USER/cgi-bin/icinga ~/opt/icinga/sbin
    ln -s /var/www/virtual/$USER/html/icinga ~/opt/icinga/share

Jetzt können wir Icinga kompilieren:

    make all
    make install
    make install-init
    make install-config
    make install-webconf

Damit suEXEC die Ausführung der .cgi-Dateien nicht verweigert, müssen deren Zugriffsrechte angepasst werden:

    chmod 755 /var/www/virtual/$USER/cgi-bin/icinga/
    chmod 755 /var/www/virtual/$USER/cgi-bin/icinga/*.cgi

Nun muss die Datei `/home/$USER/opt/icinga/etc/icinga.cfg` angepasst werden:

`use_syslog` ist dabei auf `0` zu setzen

`/home/$USER/opt/icinga/etc/init.d/icinga` benötigt ebenfalls eine Anpassung: 

`IcingaLockDir` ist dabei auf `${prefix}/var/lock/subsys` zu setzen.

Nun wird die Weboberfläche von Icinga noch gegen unbefugte Zugriffe abgesichert:

    htpasswd -s -c /var/www/virtual/$USER/html/icinga/.htpasswd icingaadmin

Folgendes in die Datei `/var/www/virtual/$USER/html/icinga/.htaccess` schreiben (UBERSPACE anpassen!):

    AuthName "Icing Access"
    AuthType Basic
    AuthUserFile /var/www/virtual/UBERSPACE/html/icinga/.htpasswd
    Require valid-user

Die CGI-Skripte von Icinga sollten auch geschützt werden:

    cp /var/www/virtual/$USER/html/icinga/.htaccess /var/www/virtual/$USER/cgi-bin/icinga/

Jetzt noch Icinga ausführbar machen:

    chmod +x ~/opt/icinga/etc/init.d/icinga 

Icinga selber ist nun fertig installiert und kann auch schon gestartet werden:

  * Icinga-Konfiguration prüfen: `~/opt/icinga/etc/init.d/icinga checkconfig`
  * Icinga starten: `~/opt/icinga/etc/init.d/icinga start`

Damit Icinga beim Neustart des Servers ebenfalls mitgestartet wird, wird ein daemontools-daemon benötigt.

### Nagios Plugins

Die ganzen Skripte mit den Prüfroutinen (`check_xyz`) müssen separat installiert werden:

    cd ~/src/
    
    wget https://www.nagios-plugins.org/download/nagios-plugins-1.4.16.tar.gz
    tar xzf nagios-plugins-1.4.16.tar.gz

    cd nagios-plugins-1.4.16/
    ./configure --prefix=/home/$USER/opt/icinga \
    --with-nagios-user=$USER \
    --with-nagios-group=$USER 

    make
    make install

### NRPE

Wer andere Server via NRPE überwachen möchte benötigt `check_nrpe`:

    cd ~/src/
    wget http://sourceforge.net/projects/nagios/files/nrpe-2.x/nrpe-2.13/nrpe-2.13.tar.gz/download -O nrpe-2.13.tar.gz
    tar xzf nrpe-2.13.tar.gz

    cd nrpe-2.13/
    ./configure --prefix=/home/$USER/opt/icinga \
    --with-nagios-user=$USER \
    --with-nagios-group=$USER \
    --with-nrpe-user=$USER \
    --with-nrpe-group=$USER

    make
    make install