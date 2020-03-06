# Instalación y configuración de {php}IPAM en Debian9/10

## Autor

- [Ixen Rodríguez Pérez - kurosaki1976](ixenrp1976@gmail.com)

## Introducción

### ¿Qué es `{php}IPAM`?

Como su nombre compuesto indica, `{php}IPAM` es una aplicación web de código abierto, para el manejo y administración por medio de lenguaje `PHP`, de las direcciones del Protocolo de Internet (en inglés `Internet Protocol Address Management`, `IPAM`). Su objetivo es proporcionar una gestión de direcciones `IP`, ligera, moderna y útil. Utiliza `backend` de base de datos `MySQL/MariaDB`, bibliotecas `jQuery`, `ajax` y características `HTML5/CSS3`.

Entre sus principales funcionalidades, que hacen olvidarnos de productos de pagos, están:

- gestión de direccionamiento `IPv4/IPv6`,
- gestión de subredes (`subnetting`),
- visualización automática de subredes disponibles,
- visualización gráfica de subredes,
- descubrimiento automático de nuevos `hosts` en subredes,
- comprobaciones automáticas del estado de `hosts`,
- autenticación de dominio (`LDAP`/Directorio Activo/`Radius`),
- gestión de `VLAN`,
- calculadora `IPv4/IPv6`.

## Aplicación web `{php}IPAM`

- Descargar `{php}IPAM`

```bash
wget https://sourceforge.net/projects/phpipam/files/phpipam-1.4.tar
tar -xvf phpipam-1.4.tar -C /opt
```

> **NOTA**: También se puede utilizar la alternativa `GitHub`.
>
> ```bash
> git clone --recursive https://github.com/phpipam/phpipam.git /opt/phpipam
> cd /opt/phpipam
> git checkout -b 1.4 origin/1.4
> ```

- Establecer permisos

```bash
chown -R www-data:www-data /opt/phpipam
find /opt/phpipam -type f \-exec chmod 644 {} \;
find /opt/phpipam -type d \-exec chmod 755 {} \;
```

- Definir parámetros de acceso a la base de datos

```bash
cp /opt/phpipam/config.dist.php /opt/phpipam/config.php
nano /opt/phpipam/config.php

$ db ['host'] = 'localhost';
$ db ['user'] = 'phpipam_admin';
$ db ['pass'] = 'My$ecr3tP@s$w0rd.';
$ db ['name'] = 'phpipam_db';
$ db ['port'] = 3306;
```

## Gestor de base de datos `MariaDB`

- Instalar paquetes necesarios

```bash
apt install mariadb-server mariadb-client
```

> **NOTA**: Después de instalado el gestor de bases de datos, se deben establecer mecanismos de seguridad necesarios para el entorno de trabajo de `MariaDB` (establecer contraseña del usuario `root`, etc.), ejecutando `mysql_secure_installation`.

- Crear base de datos e inicializarla

```bash
mysql -u root -p

MariaDB [(none)]> CREATE DATABASE phpipam_db;
MariaDB [(none)]> GRANT ALL ON phpipam_db.* TO 'phpipam_admin'@'localhost' IDENTIFIED BY 'My$ecr3tP@s$w0rd.';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> QUIT;

mysql -u phpipam_admin -p phpipam_db < /opt/db/SCHEMA.sql
```

## Servidor web (`Apache2/Nginx`)

- Crear certificado de seguridad TLS/SSL

```bash
openssl req -x509 -nodes -days 3650 -sha512 \
    -subj "/C=CU/ST=Provincia/L=Ciudad/O=EXAMPLE TLD/OU=IT/CN=phpipam.example.tld/emailAddress=postmaster@example.tld/" \
    -newkey rsa:4096 \
    -out /etc/ssl/certs/phpIPAM.crt \
    -keyout /etc/ssl/private/phpIPAM.key

openssl dhparam -out /etc/ssl/dh2048.pem 2048
chmod 0444 /etc/ssl/certs/phpIPAM.crt
chmod 0400 /etc/ssl/private/phpIPAM.key
```

### `Apache`

- Instalar paquetes necesarios

```bash
apt install apache2 php-cli libapache2-mod-php php-curl php-mysql php-curl php-gd php-intl \
    php-pear php-imap php-memcache php-pspell php-recode php-tidy php-xmlrpc php-mbstring \
    php-gettext php-gmp php-json php-xml php-ldap php-mcrypt php-snmp
```

- Definir zona horaria

```bash
sed -i "s/^;date\.timezone =.*$/date\.timezone = 'America\/Havana'/;
    s/^;cgi\.fix_pathinfo=.*$/cgi\.fix_pathinfo = 0/" \
    /etc/php/7*/cli/php.ini
```

- Crear `VirtualHost`

```bash
nano /etc/apache2/sites-available/phpIPAM.conf

<VirtualHost *:80>
    RewriteEngine on
    RewriteCond %{HTTPS} =off
    RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [QSA,L,R=301]
</VirtualHost>
<IfModule mod_ssl.c>
    <VirtualHost phpipam.example.tld:443>
        ServerName phpipam.example.tld
        ServerAdmin postmaster@example.tld
        DirectoryIndex index.php
        DocumentRoot /opt/phpipam
        <Directory "/opt/phpipam">
            Options Indexes FollowSymLinks MultiViews
            AllowOverride All
            Require all granted
        </Directory>
        SSLEngine on
        SSLCertificateFile /etc/ssl/certs/phpIPAM.crt
        SSLCertificateKeyFile /etc/ssl/private/phpIPAM.key
        <FilesMatch "\.(cgi|shtml|phtml|php)$">
            SSLOptions +StdEnvVars
        </FilesMatch>
        <Directory /usr/lib/cgi-bin>
            SSLOptions +StdEnvVars
        </Directory>
        BrowserMatch "MSIE [2-6]" nokeepalive ssl-unclean-shutdown downgrade-1.0 force-response-1.0
        BrowserMatch "MSIE [17-9]" ssl-unclean-shutdown
        ErrorLog ${APACHE_LOG_DIR}/phpIPAM_error.log
        CustomLog ${APACHE_LOG_DIR}/phpIPAM_access.log combined
    </VirtualHost>
</IfModule>
```

- Activar módulos necesarios, `VirtualHost` y reiniciar servidor web

```bash
a2enmod rewrite ssl
a2ensite /etc/apache2/sites-available/phpIPAM.conf
systemctl restart apache2.service
```

> **NOTA**: Los ficheros de bitácora de acceso (`phpIPAM_access.log`) y errores (`phpIPAM_error.log`), se almacenan en `/var/logs/apache2/`. Para monitorear sus respectivas salidas, ejecutar `tail -fn100 /var/logs/apache2/NOMBRE_FICHERO_LOG`.

### `Nginx`

- Instalar paquetes necesarios

```bash
apt install nginx-full php-fpm php-curl php-mysql php-curl php-gd php-intl php-pear \
    php-imap php-memcache php-pspell php-recode php-tidy php-xmlrpc php-mbstring \
    php-gettext php-gmp php-json php-xml php-ldap php-mcrypt php-snmp
```

- Definir zona horaria

```bash
sed -i "s/^;date\.timezone =.*$/date\.timezone = 'America\/Havana'/;
    s/^;cgi\.fix_pathinfo=.*$/cgi\.fix_pathinfo = 0/" \
    /etc/php/7*/fpm/php.ini
```

- Crear `VirtualHost`

```bash
nano /etc/nginx/sites-available/phpIPAM.conf

proxy_cache_path /tmp/cache keys_zone=cache:10m levels=1:2 inactive=600s max_size=100m;
limit_req_zone $binary_remote_addr zone=download:10m rate=2r/s;
server {
    listen 80;
    listen 443 ssl http2;
    root /opt/phpipam;
    index index.php;
    server_name phpipam.example.tld;
    if ($scheme = http) {
        return 301 https://$server_name$request_uri;
    }
    ssi on;
    ssl_certificate /etc/ssl/certs/phpIPAM.crt;
    ssl_certificate_key /etc/ssl/private/phpIPAM.key;
    charset utf-8;
    ssl_dhparam /etc/ssl/dh2048.pem;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
    ssl_ecdh_curve secp384r1;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;
    ssl_stapling_verify on;
    resolver 127.0.0.1 valid=300s;
    resolver_timeout 5s;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
    add_header X-Content-Type-Options nosniff;
    proxy_cache cache;
    proxy_cache_valid 200 1s;
    location ~ [^/]\.php(/|$) {
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        if (!-f $document_root$fastcgi_script_name) {
            return 404;
        }
        fastcgi_param HTTP_PROXY "";
        fastcgi_pass unix:/var/run/php/php7*-fpm.sock;
        include snippets/fastcgi-php.conf;
    }
    location ~ /\. {
        deny all;
    }
    location = /favicon.ico {
        log_not_found off;
        access_log off;
    }
    location = /robots.txt {
        access_log off;
        log_not_found off;
    }
    location /phpipam/ {
        try_files $uri $uri/ /phpipam/index.php;
        index index.php;
    }
    location /phpipam/api/ {
        try_files $uri $uri/ /phpipam/api/index.php;
    }
    access_log /var/log/nginx/phpIPAM_access.log;
    error_log /var/log/nginx/phpIPAM_error.log;
}
```

- Habilitar `VirtualHost` y reiniciar servidor web

```bash
ln -s /etc/nginx/sites-available/phpIPAM.conf /etc/nginx/sites-enabled/
systemctl restart php7*-fpm.service nginx.service
```

> **NOTA**: Los ficheros de bitácora de acceso (`phpIPAM_access.log`) y errores (`phpIPAM_error.log`), se almacenan en `/var/logs/nginx/`. Para monitorear sus respectivas salidas, ejecutar `tail -fn100 /var/logs/nginx/NOMBRE_FICHERO_LOG`.

## Acceder a `{php}IPAM`

Finalmente, acceder a la aplicación web, tecleando la `URL` [phpipam.example.tld](https://phpipam.example.tld/), en el navegador de preferencia y usar el par usuario/contraseña (`admin/ipamadmin`) para efectuar el `login`.

## Conclusiones

La explotación del sistema, si se es administrador de red, o se tienen conocimientos de redes, direccionamiento `IPv4/IPv6`, `VLANs`, `DNS`, etc.; no es complicada. No obstante, en las referencias, se listan enlaces que pueden servir de ayuda o realizar búsquedas en Internet acerca del tema.

Sin dudas, `{php}IPAM` es una alternativa fabulosa ante las muchas opciones privativas existentes, desde `Infloblox` hasta la característica `IPAM` de `Microsoft` disponible a partir de `Microsoft Windows Server 2008` y posteriores.

## Referencias

* [phpIPAM Open-source IP address management](https://phpipam.net/)
* [phpipam installation guide](https://phpipam.net/documents/installation/)
* [phpIPAM on nginx](https://phpipam.net/news/phpipam-on-nginx/)
* [Reset phpipam admin password](https://phpipam.net/news/reset-phpipam-admin-password/)
* [¿Cómo usar phpIPAM como herramienta auxiliar en Pandora FMS?](https://pandorafms.com/blog/es/phpipam/)
* [Tag: phpIPAM](https://www.jorgedelacruz.es/tag/phpipam/)
* [Installing phpIPAM on Ubuntu 16.04](https://ithinkvirtual.com/2016/05/08/installing-phpipam-on-ubuntu-16-04/)
* [Instalar phpIPAM en Debian 9 con Nginx y MariaDB](https://www.sysadminsdecuba.com/2018/11/instalar-phpipam-en-debian-9-con-nginx-y-mariadb/)
* [Best Free & Paid IPAM Tools for IP Address Tracking & Management](https://www.ittsystems.com/best-ipam-tools-ip-address-tracking-management/)
