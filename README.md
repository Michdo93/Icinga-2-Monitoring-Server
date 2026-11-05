# Icinga 2 Monitoring-Server -- Installation und Betrieb

Diese Dokumentation beschreibt die erfolgreich getestete Installation
eines einzelnen Icinga-Monitoring-Servers auf **Debian 13 (Trixie)**
mit:

-   Icinga 2
-   Icinga 2 API
-   Icinga DB
-   Redis (`icingadb-redis`)
-   MariaDB
-   Icinga Web 2
-   Icinga DB Web
-   Icinga Director
-   Apache 2 mit HTTPS
-   Smallstep `step-ca` als interne Zertifizierungsstelle
-   ACME/Certbot für das HTTPS-Zertifikat von Apache
-   automatischer Zertifikatserneuerung für Apache und die Icinga-API

Die Anleitung bildet den tatsächlich eingerichteten Einzelserver ab. Sie
enthält bewusst nicht die zwischenzeitlichen Fehlversuche und
Bereinigungsarbeiten aus der Shell-History.

> **Wichtig:** Passwörter, private Schlüssel und andere Geheimnisse
> gehören nicht in dieses README oder in ein Git-Repository. Alle
> Platzhalter wie `<PASSWORT>` müssen lokal durch eigene Werte ersetzt
> werden. Bereits vergebene Passwörter werden nicht erneut erzeugt oder
> geändert, nur weil sie in einem Beispiel vorkommen.

------------------------------------------------------------------------

## 1. Zielarchitektur

### 1.1 Komponenten und Datenfluss

``` text
                         Browser
                            |
                         HTTPS :443
                            |
                         Apache
                            |
                      Icinga Web 2
                       /         \
                      /           \
              Icinga DB Web     Director
                  |                |
                  |                | Icinga 2 API :5665
                  |                |
             MariaDB           Icinga 2
                  ^                |
                  |                |
               Icinga DB <---------+
                  |
                Redis
```

Icinga 2 führt die Checks aus und stellt seinen Zustand über die API
bereit. Icinga DB synchronisiert Konfigurationen und Zustände und
schreibt die Daten in Redis und MariaDB. Icinga DB Web liest diese Daten
und stellt sie in Icinga Web 2 dar. Der Director verwaltet
Icinga-Objekte und veröffentlicht Änderungen über die Icinga-API.

### 1.2 Server- und Netzwerkinformationen

  -------------------------------------------------------------------------------------
  Eigenschaft                         Wert
  ----------------------------------- -------------------------------------------------
  Betriebssystem                      Debian 13 (Trixie)

  Hostname / FQDN                     `icinga.home.arpa`

  Interne DNS-Zone                    `home.arpa`

  HTTPS                               TCP 443

  HTTP / ACME HTTP-01                 TCP 80

  Icinga 2 API                        TCP 5665

  MariaDB                             TCP 3306, lokal

  Redis                               TCP 6380, lokal

  Smallstep CA                        `https://step-ca.home.arpa`

  Smallstep ACME-Verzeichnis          `https://step-ca.home.arpa/acme/acme/directory`
  -------------------------------------------------------------------------------------

Der Server verwendet einen internen DNS-Eintrag für `icinga.home.arpa`.
Die Namensauflösung muss auf den Icinga-Server zeigen, bevor Zertifikate
ausgestellt und die Weboberfläche verwendet werden.

### 1.3 Zertifikatsstrategie

Es werden zwei getrennte Zertifikatsverwendungen eingesetzt:

1.  **Apache / Icinga Web 2:** Zertifikat über ACME und Certbot. Die
    Zertifikatsdateien liegen unter
    `/etc/letsencrypt/live/icinga.home.arpa/`.
2.  **Icinga 2 API:** Eigenes Zertifikat von Smallstep mit
    `CN=icinga.home.arpa`, passendem SAN sowie `serverAuth` und
    `clientAuth`. Dieses Zertifikat liegt unter
    `/var/lib/icinga2/certs/`.

Das Apache-Zertifikat wird nicht einfach für die Icinga-API
wiederverwendet. Das separate API-Zertifikat wird mit dem Step-CLI
ausgestellt und hat die für Icinga erforderliche Identität und
Verwendungszwecke.

------------------------------------------------------------------------

## 2. Voraussetzungen

Vor der Installation müssen folgende Voraussetzungen erfüllt sein:

-   Debian 13 ist installiert und aktualisiert.
-   Der Server besitzt einen festen DNS-Namen `icinga.home.arpa`.
-   `icinga.home.arpa` wird im internen DNS auf die IP-Adresse dieses
    Servers aufgelöst.
-   Die Smallstep-CA ist unter `https://step-ca.home.arpa` erreichbar.
-   Auf der Smallstep-CA existiert ein ACME-Provisioner mit dem Namen
    `acme`.
-   Auf der Smallstep-CA existiert ein JWK-Provisioner für die
    Ausstellung des Icinga-API-Zertifikats.
-   Die Root-CA-Fingerprint wurde über einen vertrauenswürdigen Weg
    ermittelt.
-   Der Server kann die Paketquellen von Debian und Icinga erreichen.

Verwendete Root-CA-Fingerprint:

``` text
29d36c7e3b0394e453e3b95e4059c3d20e5adf4496ef39e9a3c21a5e5ecd973f
```

Die Fingerprint muss vor dem Bootstrap mit dem tatsächlichen Wert der
eigenen CA verglichen werden. Sie ist kein Geheimnis, aber ein
sicherheitsrelevanter Vertrauensanker.

------------------------------------------------------------------------

## 3. Debian vorbereiten

Als `root` arbeiten oder Befehle mit `sudo` ausführen.

``` bash
apt update
apt full-upgrade -y
apt autoremove --purge -y
```

Grundlegende Werkzeuge installieren:

``` bash
apt install -y \
    curl \
    wget \
    gpg \
    ca-certificates \
    openssl \
    gnupg \
    apt-transport-https \
    lsb-release
```

Systemzeit und DNS-Auflösung prüfen:

``` bash
timedatectl
getent hosts icinga.home.arpa
getent hosts step-ca.home.arpa
```

Beide Namen müssen auf die jeweils erwarteten internen IP-Adressen
aufgelöst werden.

------------------------------------------------------------------------

## 4. Icinga-Paketquelle einrichten

Das offizielle Icinga-Archiv-Keyring-Paket wird von der
Icinga-Paketquelle bezogen und installiert. Die heruntergeladene Datei
muss aus einer vertrauenswürdigen offiziellen Quelle stammen.

Beispiel, wenn `icinga-archive-keyring.deb` im aktuellen Verzeichnis
liegt:

``` bash
apt install ./icinga-archive-keyring.deb
```

Paketquelle für Debian 13 (Trixie) einrichten:

``` bash
cat > /etc/apt/sources.list.d/icinga.sources <<'EOF'
Types: deb deb-src
URIs: https://packages.icinga.com/debian/
Suites: icinga-trixie
Components: main
Signed-By: /usr/share/keyrings/icinga-archive-keyring.gpg
EOF
```

Paketlisten aktualisieren:

``` bash
apt update
```

Kontrollieren, ob die Icinga-Pakete aus der erwarteten Quelle angeboten
werden:

``` bash
apt-cache policy icinga2 icingadb icingadb-redis icingadb-web icingaweb2 icinga-director
```

------------------------------------------------------------------------

## 5. MariaDB installieren und Datenbanken vorbereiten

### 5.1 Installation

``` bash
apt install -y mariadb-server
systemctl enable --now mariadb
systemctl status mariadb --no-pager
```

MariaDB wird in dieser Installation nur lokal verwendet. Der
Datenbankport muss nicht im LAN erreichbar sein.

### 5.2 Datenbanken und Benutzer

Es werden drei getrennte Datenbanken mit jeweils eigenen
Datenbankbenutzern verwendet:

  Datenbank     Benutzer      Verwendung
  ------------- ------------- -----------------------------------------
  `icingadb`    `icingadb`    Icinga-DB-Daten
  `icingaweb`   `icingaweb`   Icinga-Web-2-Konfiguration und Benutzer
  `director`    `director`    Icinga Director

Für jeden Datenbankbenutzer ein eigenes langes Passwort verwenden.
Passwörter können beispielsweise lokal erzeugt werden:

``` bash
openssl rand -hex 32
```

Die Ausgabe nicht in ein öffentliches Repository übernehmen. Passwörter
sicher lokal ablegen.

MariaDB-Konsole öffnen:

``` bash
mariadb
```

Beispiel für die Anlage der Datenbanken und Benutzer
(Passwort-Platzhalter lokal ersetzen):

``` sql
CREATE DATABASE icingadb
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

CREATE DATABASE icingaweb
  CHARACTER SET utf8
  COLLATE utf8_general_ci;

CREATE DATABASE director
  CHARACTER SET utf8
  COLLATE utf8_general_ci;

CREATE USER 'icingadb'@'localhost' IDENTIFIED BY '<PASSWORT_ICINGADB>';
CREATE USER 'icingaweb'@'localhost' IDENTIFIED BY '<PASSWORT_ICINGAWEB>';
CREATE USER 'director'@'localhost' IDENTIFIED BY '<PASSWORT_DIRECTOR>';

GRANT ALL PRIVILEGES ON icingadb.* TO 'icingadb'@'localhost';
GRANT ALL PRIVILEGES ON icingaweb.* TO 'icingaweb'@'localhost';
GRANT ALL PRIVILEGES ON director.* TO 'director'@'localhost';

FLUSH PRIVILEGES;
EXIT;
```

Falls Datenbanken oder Benutzer bereits existieren, nicht blind erneut
anlegen. Vorhandene Konten und Passwörter beibehalten oder die
SQL-Anweisungen gezielt an den Ist-Zustand anpassen.

### 5.3 Icinga-DB-Schema importieren

``` bash
mariadb icingadb < /usr/share/icingadb/schema/mysql/schema.sql
```

Kontrolle:

``` bash
mariadb -e "SHOW TABLES FROM icingadb;"
```

### 5.4 Icinga-Web-2-Schema importieren

``` bash
mariadb icingaweb < /usr/share/icingaweb2/schema/mysql.schema.sql
```

Kontrolle:

``` bash
mariadb -e "SHOW TABLES FROM icingaweb;"
```

Das Director-Schema wird später über die Director-Migrationen erzeugt.
Es wird nicht manuell aus einer beliebigen SQL-Datei importiert.

------------------------------------------------------------------------

## 6. Apache und PHP installieren

``` bash
apt install -y \
    apache2 \
    libapache2-mod-php \
    php \
    php-cli \
    php-intl \
    php-gd \
    php-mysql \
    php-curl \
    php-mbstring \
    php-xml \
    php-ldap \
    php-imagick
```

Apache aktivieren:

``` bash
systemctl enable --now apache2
```

PHP-Version prüfen:

``` bash
php -v
```

------------------------------------------------------------------------

## 7. Icinga-Pakete installieren

``` bash
apt install -y \
    icinga2 \
    monitoring-plugins \
    icingadb-redis \
    icingadb \
    icingadb-web \
    icingaweb2 \
    icingacli \
    icinga-director
```

Versionen kontrollieren:

``` bash
icinga2 --version
icingadb --version
```

Paketversionen der getesteten Installation (Stand der Einrichtung):

  Komponente        Version
  ----------------- ---------
  Icinga 2          2.16.6
  Icinga DB         1.6.0
  Icinga DB Redis   8.2.11
  Icinga DB Web     1.5.0
  Icinga Web 2      2.14.1
  Icinga Director   1.12.1
  MariaDB           11.8.6
  PHP               8.4.x

Paketversionen können sich durch spätere Updates ändern.

------------------------------------------------------------------------

## 8. Smallstep-CA auf dem Icinga-Server vertrauen

Das Step-CLI muss installiert sein. In dieser Installation wurde es
bereits auf dem Server bereitgestellt.

Bootstrap mit CA-URL und Root-Fingerprint:

``` bash
step ca bootstrap \
    --ca-url https://step-ca.home.arpa \
    --fingerprint 29d36c7e3b0394e453e3b95e4059c3d20e5adf4496ef39e9a3c21a5e5ecd973f
```

CA-Erreichbarkeit prüfen:

``` bash
step ca health
```

Provisioner anzeigen:

``` bash
step ca provisioner list \
    --ca-url https://step-ca.home.arpa \
    --root /root/.step/certs/root_ca.crt
```

ACME-Verzeichnis prüfen:

``` bash
curl --cacert /root/.step/certs/root_ca.crt \
    https://step-ca.home.arpa/acme/acme/directory
```

Root-CA in den System-Truststore aufnehmen:

``` bash
install -m 0644 \
    /root/.step/certs/root_ca.crt \
    /usr/local/share/ca-certificates/smarthome-lab-root.crt

update-ca-certificates
```

Vertrauensprüfung:

``` bash
curl -I https://step-ca.home.arpa
```

------------------------------------------------------------------------

## 9. Icinga 2 API aktivieren

API-Feature einrichten:

``` bash
icinga2 api setup
```

Node-Name und Zone kontrollieren:

``` bash
grep -E 'NodeName|ZoneName' /etc/icinga2/constants.conf
```

Für diese Installation lautet der Node-Name:

``` text
icinga.home.arpa
```

Die API-Konfiguration befindet sich unter:

``` text
/etc/icinga2/features-enabled/api.conf
```

Die API muss auf TCP-Port `5665` lauschen. Die Konfiguration und
Bind-Adresse prüfen:

``` bash
cat /etc/icinga2/features-enabled/api.conf
```

### 9.1 API-Benutzer anlegen

In `/etc/icinga2/conf.d/api-users.conf` werden getrennte API-Benutzer
für Icinga DB Web und Director angelegt.

Beispiel für den Director-Benutzer:

``` icinga2
object ApiUser "director-api" {
  password = "<LANGES_API_PASSWORT>"

  permissions = [
    "*",
    "filter-expression"
  ]
}
```

Zusätzlich wird ein API-Benutzer `icingadb-web` mit den für Icinga DB
Web vorgesehenen Berechtigungen angelegt. Die Zugangsdaten müssen mit
den späteren Angaben im Web-Setup übereinstimmen.

**Sicherheit:** API-Passwörter nicht in Git einchecken. Die
tatsächlichen Berechtigungen des `icingadb-web`-Benutzers aus der
installierten Icinga-DB-Web-Dokumentation bzw. der vorgesehenen
Konfiguration übernehmen; nicht pauschal die Director-Rechte verwenden.

Konfiguration prüfen:

``` bash
icinga2 daemon -C
```

Icinga 2 starten:

``` bash
systemctl enable --now icinga2
systemctl status icinga2 --no-pager
```

------------------------------------------------------------------------

## 10. Eigenes Smallstep-Zertifikat für die Icinga-API

### 10.1 Zertifikat ausstellen

Das API-Zertifikat wird separat von Smallstep ausgestellt. Der
JWK-Provisioner muss die Ausstellung für `icinga.home.arpa` erlauben.

``` bash
step ca certificate \
    icinga.home.arpa \
    /root/icinga-api.crt \
    /root/icinga-api.key \
    --provisioner "<JWK_PROVISIONER>" \
    --not-after 24h \
    --ca-url https://step-ca.home.arpa \
    --root /root/.step/certs/root_ca.crt
```

Der Provisioner kann eine Authentifizierung verlangen. Das
Provisioner-Passwort ausschließlich lokal eingeben und niemals in die
Dokumentation schreiben.

Zertifikat kontrollieren:

``` bash
openssl x509 \
    -in /root/icinga-api.crt \
    -noout -subject -issuer -dates -ext subjectAltName -ext extendedKeyUsage
```

Erwartet werden:

-   Subject/CN `icinga.home.arpa`
-   SAN `DNS:icinga.home.arpa`
-   Extended Key Usage für TLS Server Authentication und TLS Client
    Authentication
-   Signatur durch die SmartHome Intermediate CA

Zertifikatskette prüfen:

``` bash
openssl verify \
    -CAfile /root/.step/certs/root_ca.crt \
    /root/icinga-api.crt
```

### 10.2 Zertifikatsdateien installieren

Icinga verwendet das Benutzerkonto `nagios`. Zertifikatsverzeichnis
anlegen:

``` bash
install -d \
    -o nagios \
    -g nagios \
    -m 0750 \
    /var/lib/icinga2/certs
```

Leaf-Zertifikat und privaten Schlüssel installieren:

``` bash
install \
    -o nagios \
    -g nagios \
    -m 0644 \
    /root/icinga-api.crt \
    /var/lib/icinga2/certs/icinga.home.arpa.crt

install \
    -o nagios \
    -g nagios \
    -m 0600 \
    /root/icinga-api.key \
    /var/lib/icinga2/certs/icinga.home.arpa.key
```

Smallstep-Root-CA als Vertrauensanker für Icinga installieren:

``` bash
install \
    -o nagios \
    -g nagios \
    -m 0644 \
    /root/.step/certs/root_ca.crt \
    /var/lib/icinga2/certs/ca.crt
```

> In der hier eingerichteten Einzelserver-Konfiguration wurde die von
> `icinga2 api setup` erzeugte interne Icinga-CA nicht mehr für das
> API-Serverzertifikat verwendet. Ihr privater Schlüssel wurde gesichert
> und aus dem aktiven CA-Verzeichnis verschoben. Das Originalzertifikat
> unter `/var/lib/icinga2/ca/ca.crt` blieb erhalten. Die private
> Schlüsseldatei wurde unter `/root/icinga-pki-backup/` gesichert. Dies
> ist eine bewusste Abweichung vom Icinga-Standard-PKI-Aufbau und sollte
> bei einer späteren Erweiterung auf mehrere Icinga-Nodes erneut
> bewertet werden.

### 10.3 Icinga-Konfiguration und TLS testen

``` bash
icinga2 daemon -C
systemctl restart icinga2
systemctl status icinga2 --no-pager
```

Prüfen, ob die API lauscht:

``` bash
ss -lntp | grep ':5665'
```

TLS-Verbindung und Zertifikatskette prüfen:

``` bash
openssl s_client \
    -connect 127.0.0.1:5665 \
    -servername icinga.home.arpa \
    -CAfile /root/.step/certs/root_ca.crt \
    -verify_return_error \
    </dev/null
```

Erwartet:

``` text
Verify return code: 0 (ok)
```

------------------------------------------------------------------------

## 11. Icinga DB Redis konfigurieren

Der Paketdienst `icingadb-redis` stellt Redis lokal auf `127.0.0.1:6380`
bereit.

``` bash
systemctl enable --now icingadb-redis
systemctl status icingadb-redis --no-pager
```

Die Konfiguration von Icinga DB Redis ist paketabhängig. In der
installierten Paketversion wird der Redis-Dienst als `icingadb-redis`
betrieben und lauscht auf Port `6380`.

Redis später mit `redis-tools` prüfen:

``` bash
apt install -y redis-tools

redis-cli -h 127.0.0.1 -p 6380 PING
redis-cli -h 127.0.0.1 -p 6380 DBSIZE
```

Erwartet wird `PONG`. Die Schlüsselanzahl ist abhängig vom laufenden
System und nicht als fester Sollwert zu verstehen.

------------------------------------------------------------------------

## 12. Icinga DB konfigurieren und starten

Die Konfigurationsdatei liegt unter:

``` text
/etc/icingadb/config.yml
```

Dort werden die Verbindung zu Redis und die Verbindung zur Datenbank
`icingadb` konfiguriert. Die genauen YAML-Schlüssel anhand der mit dem
installierten Paket gelieferten Beispieldatei bzw. Dokumentation
übernehmen und die vorhandene Datei nicht blind überschreiben.

Die einzutragenden Verbindungsdaten:

  Einstellung       Wert
  ----------------- -----------------------------------
  Redis Host        `127.0.0.1`
  Redis Port        `6380`
  Redis Datenbank   `0`
  Redis TLS         deaktiviert
  SQL-Typ           MySQL/MariaDB
  SQL Host          `localhost`
  SQL Port          `3306`
  SQL-Datenbank     `icingadb`
  SQL-Benutzer      `icingadb`
  SQL-Passwort      lokal gesetztes Datenbankpasswort

Icinga DB starten:

``` bash
systemctl enable --now icingadb
systemctl status icingadb --no-pager
```

Logs prüfen:

``` bash
journalctl -u icingadb -n 50 --no-pager
```

Positive Meldungen sind unter anderem:

``` text
Starting config sync
Starting initial state sync
Updating 1 items of type host state
Updating 12 items of type service state
Finished initial state sync
Finished config sync
```

Die Zahlen entsprechen dem initialen Testsystem. Sie ändern sich, sobald
weitere Hosts und Services hinzukommen.

------------------------------------------------------------------------

## 13. Apache: HTTP-VHost und ACME-Webroot

Webroot für HTTP-01-Challenges anlegen:

``` bash
mkdir -p /var/www/html/.well-known/acme-challenge
```

Apache-Site `/etc/apache2/sites-available/icinga-http.conf` erstellen:

``` apache
<VirtualHost *:80>
    ServerName icinga.home.arpa

    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>
```

Site aktivieren und Apache neu laden:

``` bash
a2ensite icinga-http
apachectl configtest
systemctl reload apache2
```

HTTP-Erreichbarkeit prüfen:

``` bash
curl -I http://icinga.home.arpa/
```

ACME-Webroot testen:

``` bash
echo "acme-test" > /var/www/html/.well-known/acme-challenge/test.txt
curl http://icinga.home.arpa/.well-known/acme-challenge/test.txt
rm /var/www/html/.well-known/acme-challenge/test.txt
```

Die Challenge muss über HTTP erreichbar sein.

------------------------------------------------------------------------

## 14. Apache-Zertifikat mit Certbot und Smallstep ACME ausstellen

Certbot installieren:

``` bash
apt install -y certbot
```

Zertifikat über den Smallstep-ACME-Provisioner ausstellen:

``` bash
certbot certonly \
    --webroot \
    --webroot-path /var/www/html \
    --server https://step-ca.home.arpa/acme/acme/directory \
    --preferred-challenges http \
    --agree-tos \
    --register-unsafely-without-email \
    --non-interactive \
    -d icinga.home.arpa
```

Die Zertifikatsdateien befinden sich anschließend unter:

``` text
/etc/letsencrypt/live/icinga.home.arpa/
```

Wichtige Dateien:

  Datei             Bedeutung
  ----------------- -----------------------------
  `cert.pem`        Serverzertifikat
  `chain.pem`       Intermediate-Zertifikat(e)
  `fullchain.pem`   Serverzertifikat plus Kette
  `privkey.pem`     Privater Schlüssel

Zertifikat kontrollieren:

``` bash
openssl x509 \
    -in /etc/letsencrypt/live/icinga.home.arpa/fullchain.pem \
    -noout -subject -issuer -dates -ext subjectAltName
```

------------------------------------------------------------------------

## 15. Apache auf HTTPS umstellen

Benötigte Apache-Module aktivieren:

``` bash
a2enmod ssl
a2enmod rewrite
```

Standard-Site deaktivieren:

``` bash
a2dissite 000-default.conf
```

`/etc/apache2/sites-available/icinga-http.conf` durch folgende
Konfiguration ersetzen:

``` apache
<VirtualHost *:80>
    ServerName icinga.home.arpa

    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    # ACME HTTP-01 muss weiterhin über HTTP erreichbar sein.
    RewriteEngine On
    RewriteCond %{REQUEST_URI} !^/\.well-known/acme-challenge/
    RewriteRule ^ https://icinga.home.arpa%{REQUEST_URI} [R=301,L]
</VirtualHost>

<VirtualHost *:443>
    ServerName icinga.home.arpa

    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/icinga.home.arpa/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/icinga.home.arpa/privkey.pem

    SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1

    ErrorLog ${APACHE_LOG_DIR}/icinga-ssl-error.log
    CustomLog ${APACHE_LOG_DIR}/icinga-ssl-access.log combined
</VirtualHost>
```

Konfiguration testen und laden:

``` bash
apachectl configtest
apachectl -S
systemctl reload apache2
```

HTTP-Redirect und HTTPS prüfen:

``` bash
curl -I http://icinga.home.arpa/
curl -I https://icinga.home.arpa/
```

Erwartet:

-   HTTP leitet auf HTTPS um.
-   HTTPS liefert eine erfolgreiche HTTP-Antwort.
-   Die ACME-Challenge bleibt über HTTP erreichbar.

Zertifikat, das Apache tatsächlich ausliefert, prüfen:

``` bash
openssl s_client \
    -connect icinga.home.arpa:443 \
    -servername icinga.home.arpa \
    -CAfile /root/.step/certs/root_ca.crt \
    -verify_return_error \
    </dev/null
```

------------------------------------------------------------------------

## 16. Apache-Zertifikat automatisch erneuern

Die ACME-Zertifikate der verwendeten Smallstep-CA haben eine kurze
Laufzeit. Certbot wird deshalb regelmäßig ausgeführt. Die Erneuerung
erfolgt nur dann, wenn Certbot sie für erforderlich hält.

### 16.1 Deploy-Hook

Nach erfolgreicher Erneuerung muss Apache das neue Zertifikat laden.

``` bash
mkdir -p /etc/letsencrypt/renewal-hooks/deploy
```

Datei `/etc/letsencrypt/renewal-hooks/deploy/reload-apache`:

``` sh
#!/bin/sh
set -eu

systemctl reload apache2

logger -t certbot-deploy \
    "Apache reloaded after successful certificate renewal."
```

Berechtigungen setzen:

``` bash
chmod 750 /etc/letsencrypt/renewal-hooks/deploy/reload-apache
chown root:root /etc/letsencrypt/renewal-hooks/deploy/reload-apache
```

### 16.2 Certbot-Timer anpassen

Timer-Override anlegen:

``` bash
mkdir -p /etc/systemd/system/certbot.timer.d
```

Datei `/etc/systemd/system/certbot.timer.d/override.conf`:

``` ini
[Timer]
OnCalendar=
OnCalendar=*-*-* *:0/30:00
RandomizedDelaySec=0
AccuracySec=1s
Persistent=true
```

Systemd neu laden und Timer neu starten:

``` bash
systemctl daemon-reload
systemctl restart certbot.timer
systemctl enable certbot.timer
systemctl status certbot.timer --no-pager
systemctl list-timers certbot.timer
```

Der Timer startet Certbot alle 30 Minuten. Das bedeutet nicht, dass alle
30 Minuten ein neues Zertifikat ausgestellt wird. Certbot prüft
lediglich, ob eine Erneuerung erforderlich ist.

### 16.3 Erneuerung testen

Ein erzwungener Test stellt tatsächlich ein neues Zertifikat aus. Dies
nicht unnötig häufig durchführen:

``` bash
certbot renew \
    --cert-name icinga.home.arpa \
    --force-renewal \
    --no-random-sleep-on-renew
```

Danach prüfen:

``` bash
certbot certificates

openssl x509 \
    -in /etc/letsencrypt/live/icinga.home.arpa/cert.pem \
    -noout -subject -issuer -dates
```

Deploy-Hook prüfen:

``` bash
journalctl -t certbot-deploy -n 10 --no-pager
```

------------------------------------------------------------------------

## 17. Icinga-API-Zertifikat automatisch erneuern

Das API-Zertifikat wird unabhängig von Certbot mit `step ca renew`
erneuert. Das Skript prüft zuerst, ob das Zertifikat erneuert werden
muss. Es installiert ein neues Zertifikat erst nach erfolgreicher
Ausstellung und Validierung.

### 17.1 Verzeichnisse

``` bash
mkdir -p /var/backups/icinga2/certs
chmod 700 /var/backups/icinga2/certs
```

### 17.2 Erneuerungsskript

Datei `/usr/local/sbin/renew-icinga-certificate`:

``` bash
#!/bin/bash

set -Eeuo pipefail
umask 077

CA_URL="https://step-ca.home.arpa"
ROOT_CA="/root/.step/certs/root_ca.crt"

CERT_DIR="/var/lib/icinga2/certs"
CERT="${CERT_DIR}/icinga.home.arpa.crt"
KEY="${CERT_DIR}/icinga.home.arpa.key"

BACKUP_DIR="/var/backups/icinga2/certs"

log() {
    echo "$(date --iso-8601=seconds) $*"
}

# Prüfen, ob das Zertifikat erneuert werden muss.
if /usr/bin/step certificate needs-renewal "$CERT"; then
    log "Certificate requires renewal."
else
    status=$?

    if [[ "$status" -eq 1 ]]; then
        log "Certificate is still valid. No renewal required."
        exit 0
    fi

    log "Certificate renewal check failed with status ${status}."
    exit "$status"
fi

TMP_DIR="$(mktemp -d /run/icinga-cert-renew.XXXXXX)"
NEW_CERT=""

cleanup() {
    [[ -z "$NEW_CERT" ]] || rm -f "$NEW_CERT"
    rm -rf "$TMP_DIR"
}
trap cleanup EXIT

# Neues Zertifikat zunächst in ein temporäres Verzeichnis schreiben.
/usr/bin/step ca renew \
    --force \
    --out "${TMP_DIR}/fullchain.pem" \
    --ca-url "$CA_URL" \
    --root "$ROOT_CA" \
    "$CERT" \
    "$KEY"

# Bundle in Leaf- und Intermediate-Zertifikat aufteilen.
awk \
    -v leaf="${TMP_DIR}/leaf.pem" \
    -v intermediate="${TMP_DIR}/intermediate.pem" '
    /-----BEGIN CERTIFICATE-----/ {
        count++
    }
    count == 1 {
        print > leaf
    }
    count == 2 {
        print > intermediate
    }
    END {
        close(leaf)
        close(intermediate)
        if (count != 2) {
            exit 1
        }
    }
' "${TMP_DIR}/fullchain.pem"

# Zertifikatskette prüfen.
openssl verify \
    -CAfile "$ROOT_CA" \
    -untrusted "${TMP_DIR}/intermediate.pem" \
    "${TMP_DIR}/leaf.pem"

# Erwarteten Hostnamen prüfen.
openssl x509 \
    -in "${TMP_DIR}/leaf.pem" \
    -noout \
    -checkhost icinga.home.arpa

# Server- und Client-Authentifizierung prüfen.
openssl x509 \
    -in "${TMP_DIR}/leaf.pem" \
    -noout \
    -ext extendedKeyUsage \
    | grep -q "TLS Web Server Authentication"

openssl x509 \
    -in "${TMP_DIR}/leaf.pem" \
    -noout \
    -ext extendedKeyUsage \
    | grep -q "TLS Web Client Authentication"

# Prüfen, ob der private Schlüssel zum neuen Zertifikat passt.
openssl x509 \
    -in "${TMP_DIR}/leaf.pem" \
    -pubkey -noout \
    > "${TMP_DIR}/cert.pub"

openssl pkey \
    -in "$KEY" \
    -pubout \
    > "${TMP_DIR}/key.pub"

cmp "${TMP_DIR}/cert.pub" "${TMP_DIR}/key.pub"

# Bisheriges Zertifikat sichern.
mkdir -p "$BACKUP_DIR"
chmod 700 "$BACKUP_DIR"

BACKUP="${BACKUP_DIR}/icinga.home.arpa.$(date -u +%Y%m%dT%H%M%SZ).crt"

cp -a "$CERT" "$BACKUP"

# Neues Fullchain-Zertifikat atomar installieren.
NEW_CERT="$(mktemp "${CERT_DIR}/.icinga.home.arpa.XXXXXX")"

install \
    -o nagios \
    -g nagios \
    -m 0644 \
    "${TMP_DIR}/fullchain.pem" \
    "$NEW_CERT"

mv -f "$NEW_CERT" "$CERT"
NEW_CERT=""

# Icinga neu starten, damit das neue Zertifikat geladen wird.
if ! systemctl restart icinga2; then
    log "Icinga restart failed. Restoring previous certificate."

    cp "$BACKUP" "${CERT}.rollback"
    chown nagios:nagios "${CERT}.rollback"
    chmod 0644 "${CERT}.rollback"
    mv -f "${CERT}.rollback" "$CERT"

    systemctl restart icinga2 || true
    exit 1
fi

# Tatsächlich ausgeliefertes API-Zertifikat prüfen.
if ! timeout 15 openssl s_client \
    -connect 127.0.0.1:5665 \
    -servername icinga.home.arpa \
    -CAfile "$ROOT_CA" \
    -verify_return_error \
    </dev/null \
    > "${TMP_DIR}/tls-check.txt" 2>&1
then
    log "TLS verification failed. Restoring previous certificate."

    cp "$BACKUP" "${CERT}.rollback"
    chown nagios:nagios "${CERT}.rollback"
    chmod 0644 "${CERT}.rollback"
    mv -f "${CERT}.rollback" "$CERT"

    systemctl restart icinga2 || true
    exit 1
fi

if ! grep -q "Verify return code: 0 (ok)" "${TMP_DIR}/tls-check.txt"; then
    log "TLS verification result was unexpected. Restoring previous certificate."

    cp "$BACKUP" "${CERT}.rollback"
    chown nagios:nagios "${CERT}.rollback"
    chmod 0644 "${CERT}.rollback"
    mv -f "${CERT}.rollback" "$CERT"

    systemctl restart icinga2 || true
    exit 1
fi

log "Certificate successfully renewed and installed."
```

Skript absichern:

``` bash
chown root:root /usr/local/sbin/renew-icinga-certificate
chmod 750 /usr/local/sbin/renew-icinga-certificate
bash -n /usr/local/sbin/renew-icinga-certificate
```

### 17.3 systemd-Service und Timer

Datei `/etc/systemd/system/renew-icinga-certificate.service`:

``` ini
[Unit]
Description=Renew Icinga API certificate using Smallstep
Documentation=https://smallstep.com/docs/step-ca/renewal/
Wants=network-online.target
After=network-online.target icinga2.service

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/sbin/renew-icinga-certificate
```

Datei `/etc/systemd/system/renew-icinga-certificate.timer`:

``` ini
[Unit]
Description=Periodic renewal check for Icinga API certificate

[Timer]
OnBootSec=5min
OnUnitActiveSec=30min
RandomizedDelaySec=2min
Persistent=true
Unit=renew-icinga-certificate.service

[Install]
WantedBy=timers.target
```

Aktivieren:

``` bash
systemctl daemon-reload
systemctl enable --now renew-icinga-certificate.timer
```

Manueller Funktionstest:

``` bash
systemctl start renew-icinga-certificate.service
systemctl status renew-icinga-certificate.service --no-pager
journalctl -u renew-icinga-certificate.service -n 30 --no-pager
```

Wenn das Zertifikat noch gültig ist, ist folgende Meldung ein
erfolgreicher Lauf:

``` text
Certificate is still valid. No renewal required.
```

Timer kontrollieren:

``` bash
systemctl list-timers renew-icinga-certificate.timer
```

------------------------------------------------------------------------

## 18. Icinga DB Web und Icinga Web 2 konfigurieren

### 18.1 Datenbankressourcen

Die Datei `/etc/icingaweb2/resources.ini` enthält drei Ressourcen. Die
Passwörter werden lokal eingetragen und nicht in dieses README
übernommen.

Beispielstruktur:

``` ini
[Director DB]
type = "db"
db = "mysql"
host = "localhost"
dbname = "director"
username = "director"
password = "<PASSWORT_DIRECTOR>"
charset = "utf8"

[icingaweb_db]
type = "db"
db = "mysql"
host = "localhost"
port = "3306"
dbname = "icingaweb"
username = "icingaweb"
password = "<PASSWORT_ICINGAWEB>"
charset = "utf8"
use_ssl = "0"

[icingadb]
type = "db"
db = "mysql"
host = "localhost"
port = "3306"
dbname = "icingadb"
username = "icingadb"
password = "<PASSWORT_ICINGADB>"
charset = "utf8mb4"
use_ssl = "0"
```

Dateiberechtigungen:

``` bash
chown root:icingaweb2 /etc/icingaweb2/resources.ini
chmod 640 /etc/icingaweb2/resources.ini
```

Die Datei ist absichtlich nicht für `www-data` beschreibbar. Der
Webserver benötigt Leserechte über die Gruppe `icingaweb2`, aber keine
Schreibrechte auf die Zugangsdaten.

### 18.2 Icinga-DB-Web-Modul

Datei `/etc/icingaweb2/modules/icingadb/config.ini`:

``` ini
[redis]
tls = "0"

[icingadb]
resource = "icingadb"
```

Die Redis-Verbindung verwendet in dieser Paketinstallation die lokalen
Standardwerte des Moduls (Redis auf `127.0.0.1:6380`, Datenbank 0).
Falls die Paketversion oder die Redis-Konfiguration geändert wird,
müssen die Verbindungsparameter entsprechend explizit gesetzt werden.

Berechtigungen:

``` bash
chown root:icingaweb2 /etc/icingaweb2/modules/icingadb/config.ini
chmod 640 /etc/icingaweb2/modules/icingadb/config.ini
```

### 18.3 Authentication-Backend

Das Setup verwendet Datenbankauthentifizierung. Die Datei
`/etc/icingaweb2/authentication.ini` enthält sinngemäß:

``` ini
[icingaweb2]
backend = "db"
resource = "icingaweb_db"
```

Das initiale Administratorkonto der eingerichteten Instanz heißt:

``` text
sysadmin
```

Das Passwort wurde während des Web-Setups gesetzt und ist lokal
aufzubewahren.

### 18.4 Icinga Web 2 Setup-Assistent

Setup-Assistent öffnen:

``` text
https://icinga.home.arpa/icingaweb2/setup
```

Der Setup-Token kann auf dem Server erzeugt und angezeigt werden:

``` bash
icingacli setup token create
icingacli setup token show
```

Im Assistenten:

1.  Datenbank `icingaweb` und Datenbankbenutzer `icingaweb` verwenden.
2.  Datenbankauthentifizierung auswählen.
3.  Initialen Administrator `sysadmin` anlegen.
4.  Administratorgruppe einrichten.
5.  Icinga-DB-Web-Verbindung zu Redis konfigurieren:
    -   Host `127.0.0.1`
    -   Port `6380`
    -   Datenbank `0`
    -   TLS deaktiviert
    -   keine Redis-Anmeldedaten, sofern Redis lokal ohne
        Authentifizierung betrieben wird.
6.  Icinga-API-Verbindung konfigurieren:
    -   Host `icinga.home.arpa`
    -   Port `5665`
    -   API-Benutzer `icingadb-web`
    -   das lokal gesetzte API-Passwort.
7.  Module aktivieren:
    -   `director`
    -   `doc` (optional)
    -   `icingadb`

Das alte IDO-basierte Monitoring-Modul ist für diese Installation nicht
erforderlich. Die Anzeige erfolgt über Icinga DB Web.

### 18.5 Hinweis zum Ressourcen-Schreibfehler des Setup-Assistenten

Während des Setups konnte der Web-Assistent `resources.ini` nicht
schreiben, weil die Datei bewusst `root:icingaweb2` mit Modus `640`
gehört. Das ist kein Grund, die Datei für `www-data` beschreibbar zu
machen.

Stattdessen wurden die Ressourcen als `root` manuell in
`/etc/icingaweb2/resources.ini` ergänzt und die Berechtigungen
anschließend wieder auf `root:icingaweb2` und `640` gesetzt.

Nach dem manuellen Eintragen müssen die Ressourcennamen mit den
Verweisen in den Modulkonfigurationen übereinstimmen:

-   `Director DB` ↔ Director `config.ini`
-   `icingaweb_db` ↔ `authentication.ini`
-   `icingadb` ↔ Icinga-DB-Web `config.ini`

------------------------------------------------------------------------

## 19. Icinga Director einrichten

### 19.1 Modul prüfen und aktivieren

Das Modulverzeichnis liegt unter:

``` text
/usr/share/icingaweb2/modules/director/
```

Modulstatus:

``` bash
icingacli module list
```

Falls erforderlich, Director aktivieren:

``` bash
icingacli module enable director
```

Die Webserver-Gruppe muss Zugriff auf die Konfigurationsdateien haben:

``` bash
usermod -aG icingaweb2 www-data
```

Danach Apache neu starten, damit die ergänzte Gruppenzugehörigkeit
sicher übernommen wird:

``` bash
systemctl restart apache2
```

### 19.2 Director-Datenbankressource

Datei `/etc/icingaweb2/modules/director/config.ini`:

``` ini
[db]
resource = "Director DB"
```

Berechtigungen:

``` bash
chown root:icingaweb2 /etc/icingaweb2/modules/director/config.ini
chmod 640 /etc/icingaweb2/modules/director/config.ini
```

### 19.3 Kickstart-Konfiguration

Datei `/etc/icingaweb2/modules/director/kickstart.ini`:

``` ini
[config]
endpoint = icinga.home.arpa
host = icinga.home.arpa
port = 5665
username = director-api
password = "<DIRECTOR_API_PASSWORT>"
```

Berechtigungen:

``` bash
chown root:icingaweb2 /etc/icingaweb2/modules/director/kickstart.ini
chmod 640 /etc/icingaweb2/modules/director/kickstart.ini
```

### 19.4 Migrationen ausführen

Prüfen, ob Migrationen ausstehen:

``` bash
icingacli director migration pending
```

Migrationen ausführen:

``` bash
icingacli director migration run
```

### 19.5 Kickstart ausführen

Prüfen, ob Kickstart erforderlich ist:

``` bash
icingacli director kickstart required
```

Kickstart ausführen:

``` bash
icingacli director kickstart run
```

Damit übernimmt Director die grundlegenden Icinga-Objekte des
konfigurierten Endpoints und der Zone.

Objekte prüfen:

``` bash
icingacli director endpoint exists icinga.home.arpa
icingacli director zone exists icinga.home.arpa
```

### 19.6 Erstes Deployment

Deployment durchführen:

``` bash
icingacli director config deploy
```

Deployment-Status prüfen:

``` bash
icingacli director config deploymentstatus
```

Director-Gesundheit prüfen:

``` bash
icingacli director health
```

Erwartet wird:

``` text
Icinga Director: everything is fine
```

Nach erfolgreichem Deployment auch die Icinga-Konfiguration prüfen:

``` bash
icinga2 daemon -C
```

------------------------------------------------------------------------

## 20. Dienste aktivieren und Status kontrollieren

Die folgenden Dienste müssen aktiv sein:

``` bash
systemctl enable --now \
    mariadb \
    apache2 \
    icinga2 \
    icingadb-redis \
    icingadb
```

Status aller Dienste:

``` bash
systemctl status \
    icinga2 \
    icingadb \
    icingadb-redis \
    mariadb \
    apache2 \
    --no-pager
```

Kurzstatus:

``` bash
systemctl is-active \
    icinga2 \
    icingadb \
    icingadb-redis \
    mariadb \
    apache2
```

------------------------------------------------------------------------

## 21. Funktionstests

### 21.1 Icinga-Konfiguration

``` bash
icinga2 daemon -C
```

Erwartet wird eine erfolgreiche Konfigurationsvalidierung.

### 21.2 API-Port

``` bash
ss -lntp | grep ':5665'
```

### 21.3 Datenbank- und Redis-Ports

``` bash
ss -lntp | grep -E ':(3306|6380)\b'
```

MariaDB und Redis sollen in dieser Installation nur lokal erreichbar
sein.

### 21.4 Redis

``` bash
redis-cli -h 127.0.0.1 -p 6380 PING
redis-cli -h 127.0.0.1 -p 6380 DBSIZE
```

Erwartet: `PONG` und eine von null verschiedene Schlüsselanzahl, sobald
Icinga DB Daten schreibt.

### 21.5 Icinga DB

``` bash
systemctl is-active icingadb
journalctl -u icingadb -n 40 --no-pager
```

Auf erfolgreiche Konfigurations- und Zustandssynchronisation achten.

### 21.6 Director

``` bash
icingacli director health
icingacli director config deploymentstatus
icingacli director endpoint exists icinga.home.arpa
icingacli director zone exists icinga.home.arpa
```

### 21.7 HTTPS

``` bash
curl -I http://icinga.home.arpa/
curl -I https://icinga.home.arpa/
```

### 21.8 Weboberfläche

Aufrufen:

``` text
https://icinga.home.arpa/icingaweb2/
```

Mit `sysadmin` anmelden. Folgende Ansichten prüfen:

-   Taktische Übersicht: `/icingaweb2/icingadb/tactical`
-   Hosts: `/icingaweb2/icingadb/hosts`
-   Services: `/icingaweb2/icingadb/services`

In der initialen Testinstallation wurden angezeigt:

-   1 Host mit Status `UP`
-   12 Services mit Status `OK`

Die tatsächlichen Zahlen hängen von den konfigurierten Hosts und
Services ab.

------------------------------------------------------------------------

## 22. Wichtige Konfigurationsdateien

  --------------------------------------------------------------------------------------------
  Pfad                                                     Zweck
  -------------------------------------------------------- -----------------------------------
  `/etc/apt/sources.list.d/icinga.sources`                 Icinga-Paketquelle

  `/etc/icinga2/constants.conf`                            Node- und Zonenname

  `/etc/icinga2/conf.d/api-users.conf`                     Icinga-API-Benutzer

  `/etc/icinga2/features-enabled/api.conf`                 Icinga-API

  `/etc/icinga2/features-enabled/icingadb.conf`            Icinga-DB-Feature

  `/etc/icingadb/config.yml`                               Icinga-DB-Verbindungen

  `/etc/apache2/sites-available/icinga-http.conf`          HTTP-/HTTPS-VHosts

  `/etc/letsencrypt/`                                      Apache-Zertifikate und
                                                           Certbot-Konfiguration

  `/etc/letsencrypt/renewal-hooks/deploy/reload-apache`    Apache-Reload nach
                                                           Zertifikatserneuerung

  `/etc/systemd/system/certbot.timer.d/override.conf`      Certbot-Prüfintervall

  `/usr/local/sbin/renew-icinga-certificate`               Erneuerung des
                                                           Icinga-API-Zertifikats

  `/etc/systemd/system/renew-icinga-certificate.service`   API-Zertifikat-Erneuerungsdienst

  `/etc/systemd/system/renew-icinga-certificate.timer`     API-Zertifikat-Erneuerungstimer

  `/etc/icingaweb2/resources.ini`                          Datenbankressourcen

  `/etc/icingaweb2/authentication.ini`                     Web-Authentifizierung

  `/etc/icingaweb2/modules/icingadb/config.ini`            Icinga-DB-Web-Konfiguration

  `/etc/icingaweb2/modules/director/config.ini`            Director-Datenbankressource

  `/etc/icingaweb2/modules/director/kickstart.ini`         Director-Kickstart/API-Zugang
  --------------------------------------------------------------------------------------------

------------------------------------------------------------------------

## 23. Ports und Firewall

  --------------------------------------------------------------------------
  Port              Protokoll         Zweck             Erreichbarkeit
  ----------------- ----------------- ----------------- --------------------
  80                TCP               HTTP und ACME     LAN / ggf.
                                      HTTP-01           CA-Validierungsweg

  443               TCP               Icinga Web 2 über LAN
                                      HTTPS             

  5665              TCP               Icinga 2 API      Nur
                                                        vertrauenswürdige
                                                        Monitoring-Clients /
                                                        LAN

  3306              TCP               MariaDB           Nur localhost

  6380              TCP               Redis             Nur localhost
  --------------------------------------------------------------------------

MariaDB und Redis müssen nicht für andere Rechner im Netzwerk geöffnet
werden. Die Icinga-API sollte ebenfalls nur für die Systeme erreichbar
sein, die sie tatsächlich benötigen.

------------------------------------------------------------------------

## 24. Betrieb und Fehlersuche

### Dienste

``` bash
systemctl status icinga2 icingadb icingadb-redis mariadb apache2 --no-pager
```

### Icinga 2

``` bash
journalctl -u icinga2 -n 100 --no-pager
icinga2 daemon -C
```

### Icinga DB

``` bash
journalctl -u icingadb -n 100 --no-pager
```

### Redis

``` bash
journalctl -u icingadb-redis -n 100 --no-pager
redis-cli -h 127.0.0.1 -p 6380 PING
```

### Apache

``` bash
apachectl configtest
apachectl -S
journalctl -u apache2 -n 100 --no-pager
tail -n 100 /var/log/apache2/icinga-ssl-error.log
```

### Icinga Web 2

``` bash
tail -n 100 /var/log/icingaweb2/icingaweb2.log
```

### Zertifikate

Apache-Zertifikat:

``` bash
certbot certificates
openssl x509 \
    -in /etc/letsencrypt/live/icinga.home.arpa/cert.pem \
    -noout -subject -issuer -dates
```

Icinga-API-Zertifikat:

``` bash
openssl x509 \
    -in /var/lib/icinga2/certs/icinga.home.arpa.crt \
    -noout -subject -issuer -dates -ext subjectAltName
```

Timer:

``` bash
systemctl list-timers certbot.timer renew-icinga-certificate.timer
```

------------------------------------------------------------------------

## 25. Backups und Wiederherstellung

Für eine belastbare Sicherung müssen mindestens folgende Daten
berücksichtigt werden:

1.  MariaDB-Datenbanken `icingadb`, `icingaweb` und `director`.
2.  `/etc/icinga2/`
3.  `/etc/icingadb/`
4.  `/etc/icingaweb2/`
5.  `/etc/apache2/sites-available/icinga-http.conf`
6.  `/etc/letsencrypt/` einschließlich Renewal-Konfiguration.
7.  `/usr/local/sbin/renew-icinga-certificate`
8.  Die systemd-Unit- und Timer-Dateien für die Zertifikatserneuerung.
9.  Smallstep-Root-Zertifikat und die für den Betrieb benötigten
    Bootstrap-Informationen.
10. Sicher verwahrte private Schlüssel und Zugangsdaten.

Beispiel für einen logischen MariaDB-Dump:

``` bash
mariadb-dump \
    --single-transaction \
    --routines \
    --triggers \
    icingadb icingaweb director \
    > /root/icinga-databases.sql
```

Der Dump enthält Monitoring- und Konfigurationsdaten und muss geschützt
aufbewahrt werden. Für produktive Sicherungen sollten Datenbank-Dumps,
Konfigurationen und Schlüssel regelmäßig auf ein separates Backup-Ziel
übertragen und Wiederherstellungen getestet werden.

Private Schlüssel und Passwörter besonders schützen. Ein Backup der
Smallstep-Root-CA ist nicht gleichbedeutend mit einem Backup des
CA-Servers selbst; die CA-Serverdaten müssen nach dem dafür vorgesehenen
Smallstep-Backupverfahren gesichert werden.

------------------------------------------------------------------------

## 26. Ergebnis der Installation

Die Installation wurde erfolgreich getestet. Zum Zeitpunkt der Abnahme
waren:

-   Icinga 2 aktiv.
-   Icinga API auf Port 5665 erreichbar.
-   Das API-Zertifikat über Smallstep validierbar.
-   Apache über HTTPS erreichbar.
-   HTTP auf HTTPS umgeleitet.
-   Certbot-Erneuerung und Apache-Deploy-Hook eingerichtet und getestet.
-   Automatische Erneuerung des Icinga-API-Zertifikats über systemd
    eingerichtet.
-   MariaDB aktiv.
-   Redis auf `127.0.0.1:6380` erreichbar (`PONG`).
-   Icinga DB aktiv und synchronisierte einen Host sowie zwölf Services.
-   Icinga Web 2 Anmeldung mit `sysadmin` möglich.
-   Icinga DB Web zeigte die taktische Übersicht, Hosts und Services
    korrekt an.
-   Icinga Director meldete `everything is fine`.
-   Director-Deployment war erfolgreich und hatte keine ausstehenden
    Änderungen.

Damit ist die technische Basisinstallation abgeschlossen. Die Aufnahme
weiterer Netzwerkgeräte, Hostgruppen, Servicegruppen, Checks,
Benachrichtigungen und die Automatisierung über Ansible sind
nachgelagerte Konfigurationsarbeiten und nicht Teil dieser
Basisinstallation.
