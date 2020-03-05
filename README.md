# Instalación y configuración de {php}IPAM en Debian9/10

## Autor

- [Ixen Rodríguez Pérez - kurosaki1976](ixenrp1976@gmail.com)

## Introducción

### ¿Qué es {php}IPAM?

Como su nombre compuesto indica, `phpIPAM` es una aplicación web de código abierto, para el manejo y administración por medio de lenguaje `PHP`, de las direcciones del Protocolo de Internet (en inglés `Internet Protocol Address Managament`, `IPAM`). Su objetivo es proporcionar una gestión de direcciones `IP`, ligera, moderna y útil. Utiliza `backend` de base de datos `MySQL`, bibliotecas `jQuery`, `ajax` y características `HTML5/CSS3`.

Entre sus principales funcionalidades, que hacen olvidarnos de productos de pagos, están:

- gestión de direccionamiento IPv4/IPv6,
- gestión de subredes,
- visualización automática de subredes disponibles,
- visualización gráfica de subredes,
- descubrimiento automático de nuevos hosts en subredes,
- comprobaciones automáticas del estado de hosts,
- autenticación de dominio (LDAP/Directorio Activo/Radius),
- gestión de VLAN,
- calculadora IPv4/IPv6.

## Instalar y configurar {php}IPAM

- Descargar {php}IPAM

```bash
wget https://sourceforge.net/projects/phpipam/files/phpipam-1.4.tar

tar -xvf phpipam-1.4.tar -C /opt
```

- Gestor de base de datos

```bash
apt install mariadb-server mariadb-client
```

> **NOTA**: Después de instalado el gestor de bases de datos, se deben establecer mecanismos de seguridad necesarios para el entorno de trabajo de `MariaDB` (establecer contraseña del usuario `root`, etc.), ejecutando `mysql_secure_installation`.

- Crear base de datos e inicializarla

```bash
mysql -u root -p

MariaDB [(none)]> CREATE DATABASE phpipam_db;
MariaDB [(none)]> GRANT ALL ON phpipam_db.* TO 'phpipam_admin'@'localhost' IDENTIFIED BY 'My4ecre3tP@s$w0rd.';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> QUIT;

mysql -u root -p phpipam_db < /opt/db/SCHEMA.sql
```

- Servidor web y zona horaria

    #### `Apache`

    ```bash
    apt install apache2 php-cli libapache2-mod-php php-curl php-mysql php-curl php-gd php-intl php-pear php-imap php-memcache php-pspell php-recode php-tidy php-xmlrpc php-mbstring php-gettext php-gmp php-json php-xml php-ldap php-mcrypt
    ```

    ```bash
    sed -i "s/^;date\.timezone =.*$/date\.timezone = 'America\/Havana'/;
        s/^;cgi\.fix_pathinfo=.*$/cgi\.fix_pathinfo = 0/" \
        /etc/php/7*/cli/php.ini
    ```

    #### `Nginx`

    ```bash
    apt install nginx-full php-fpm php-curl php-mysql php-curl php-gd php-intl php-pear php-imap php-memcache php-pspell php-recode php-tidy php-xmlrpc php-mbstring php-gettext php-gmp php-json php-xml php-ldap php-mcrypt
    ```

    ```bash
    sed -i "s/^;date\.timezone =.*$/date\.timezone = 'America\/Havana'/;
        s/^;cgi\.fix_pathinfo=.*$/cgi\.fix_pathinfo = 0/" \
        /etc/php/7*/fpm/php.ini
    ```

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

## Referencias

* [phpIPAM Open-source IP address management](https://phpipam.net/)
* [phpIPAM on nginx](https://phpipam.net/news/phpipam-on-nginx/)
* [¿Cómo usar phpIPAM como herramienta auxiliar en Pandora FMS?](https://pandorafms.com/blog/es/phpipam/)
* [Tag: phpIPAM](https://www.jorgedelacruz.es/tag/phpipam/)
