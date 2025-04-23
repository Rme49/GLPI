#!/bin/bash

# Mettre à jour
apt update && apt upgrade -y

# Installer les paquets nécessaires
apt-get install -y apache2 php mariadb-server
apt-get install -y php-xml php-common php-json php-mysql php-mbstring php-curl php-gd php-intl php-zip php-bz2 php-imap php-apcu php-ldap

# Préconfigurer les réponses pour mysql_secure_installation
debconf-set-selections <<EOF
mysql-server mysql-server/root_password password T78952+ai
mysql-server mysql-server/root_password_again password T78952+ai
EOF

# Sécuriser l'installation de MariaDB
echo -e "\nY\nT78952+ai\nT78952+ai\nY\nY\nY\nY" | mysql_secure_installation

# Créer la base de données et l'utilisateur pour GLPI
mysql -u root -pT78952+ai <<EOF
CREATE DATABASE db_glpi;
GRANT ALL PRIVILEGES ON db_glpi.* TO glpi_adm@localhost IDENTIFIED BY "T78952+ai";
FLUSH PRIVILEGES;
EXIT
EOF

# Télécharger et extraire GLPI
cd /tmp
wget https://github.com/glpi-project/glpi/releases/download/10.0.18/glpi-10.0.18.tgz
tar -xzvf glpi-10.0.18.tgz -C /var/www/

# Configurer les permissions
chown -R www-data /var/www/glpi/
mkdir /etc/glpi
chown www-data /etc/glpi/
mv /var/www/glpi/config /etc/glpi
mkdir /var/lib/glpi
chown -R www-data /var/lib/glpi/
mv /var/www/glpi/files /var/lib/glpi
mkdir /var/log/glpi
chown www-data /var/log/glpi

# Configurer GLPI
cat << 'EOF' > /var/www/glpi/inc/downstream.php
<?php
define('GLPI_CONFIG_DIR', '/etc/glpi/');
if (file_exists(GLPI_CONFIG_DIR . '/local_define.php')) {
   require_once GLPI_CONFIG_DIR . '/local_define.php';
}
EOF

cat << 'EOF' > /etc/glpi/local_define.php
<?php
define('GLPI_VAR_DIR', '/var/lib/glpi/files');
define('GLPI_LOG_DIR', '/var/log/glpi');
EOF

apt install openssl -Y
mkdir -p /etc/ssl/glpi

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
   -keyout /etc/ssl/glpi/glpi.key \
   -out /etc/ssl/glpi/glpi.crt \
   -subj "/C=FR/ST=France/L=MDM/O=TSSR/OU=IT/CN=gleuhpi"

# Configurer Apache
cat << 'EOF' > /etc/apache2/sites-available/glpi.tssr.fr.conf
<VirtualHost *:80>
    ServerName _default_

    RewriteEngine On
    RewriteCond %{HTTPS} off
    RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
</VirtualHost>

<VirtualHost *:443>
   ServerName glpi.tssr.fr
   DocumentRoot /var/www/glpi/public

   SSLEngine on
   SSLCertificateFile /etc/ssl/glpi/glpi.crt
   SSLCertificateKeyFile /etc/ssl/glpi/glpi.key

   <Directory /var/www/glpi/public>
       Require all granted
       RewriteEngine On
       RewriteCond %{REQUEST_FILENAME} !-f
       RewriteRule ^(.*)$ index.php [QSA,L]
   </Directory>
   <FilesMatch \.php$>
       SetHandler "proxy:unix:/run/php/php8.2-fpm.sock|fcgi://localhost/"
   </FilesMatch>
</VirtualHost>
EOF

a2ensite glpi.tssr.fr.conf
a2dissite 000-default.conf
a2enmod rewrite
a2enmod ssl
a2dissite default-ssl.conf
systemctl restart apache2

# Installer et configurer PHP-FPM
apt-get install -y php8.2-fpm
a2enmod proxy_fcgi setenvif
a2enconf php8.2-fpm
systemctl reload apache2

# Configurer PHP
sed -i 's/^session.cookie_httponly =/session.cookie_httponly = on/' /etc/php/8.2/fpm/php.ini
sed -i 's/^;session.cookie_secure =/session.cookie_secure = on/' /etc/php/8.2/fpm/php.ini
systemctl restart php8.2-fpm.service
systemctl restart apache2

# Sécuriser Apache
cat << 'EOF' > /etc/apache2/conf-available/security.conf
ServerTokens Prod
ServerSignature Off
EOF


systemctl reload apache2.service
systemctl restart apache2
