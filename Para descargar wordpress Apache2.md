Para descargar wordpress:



* **Nota importante:**

  * **Cuando utilices el "nano", para guardar usa crl+o luego enter y luego crl+x**



1. Actualizar el sistema

sudo apt update && sudo apt upgrade -y



2. Instalar Apache

sudo apt install apache2 -y

sudo systemctl enable apache2

sudo systemctl start apache2



3. Instalar PHP y las extensiones que WordPress necesita

sudo apt install php libapache2-mod-php php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip php-imagick -y



4. Instalar MariaDB (la base de datos)

sudo apt install mariadb-server -y

sudo systemctl enable mariadb

sudo systemctl start mariadb



**-Luego asegura la instalación:**

sudo mysql_secure_installation



**Te va a hacer varias preguntas. Responde así:**

* Switch to unix_socket authentication? → Y
* Change the root password? → N
* Remove anonymous users? → Y
* Disallow root login remotely? → Y
* Remove test database? → Y
* Reload privilege tables? → Y



**5. Crear la base de datos para WordPress**

sudo mysql



**Dentro del prompt de MariaDB, ejecuta esto:**

CREATE DATABASE El_nombre_de_tu_base_de_datos DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE USER 'Tu_usuario'@'localhost' IDENTIFIED BY 'Tu_contraseña_segura';

GRANT ALL PRIVILEGES ON El_nombre_de_tu_base_de_datos.* TO 'Tu_usuario'@'localhost';

FLUSH PRIVILEGES;

EXIT;



**6. Descargar e instalar WordPress**

cd /tmp

wget https://wordpress.org/latest.tar.gz

tar -xzf latest.tar.gz

sudo mv wordpress /var/www/html/WordPress



**7. Permisos correctos**

sudo chown -R www-data:www-data /var/www/html/WordPress

sudo chmod -R 755 /var/www/html/WordPress



**8. Configurar Apache para WordPress**

sudo nano /etc/apache2/sites-available/WordPress.conf



**Pega esto dentro del archivo:**

--------------------------------------------------------

<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/WordPress

    <Directory /var/www/html/WordPress>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

--------------------------------------------------------



**9. Activar el sitio y el módulo rewrite**

sudo a2ensite WordPress.conf

sudo a2dissite 000-default.conf

sudo a2enmod rewrite

sudo systemctl restart apache2



**10. Finalizar en el navegador**

Ingresa a tu IP publica o a tu dominio el cual ya configuraste apuntando a tu maquina









**Opcional si tienes cloudflare túnel**

**Después de instalar, ejecuta esto en tu VM para decirle a WordPress que está detrás de un proxy HTTPS:**

sudo nano /var/www/html/wordpress/wp-config.php



**Busca la línea que dice:**

/* That's all, stop editing! Happy publishing.*/



**Y justo encima de esa línea pega esto:**

define('FORCE_SSL_ADMIN', true);

if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {

   $_SERVER['HTTPS'] = 'on';

}







**Si quieres que sea de forma global, ya que vas a tener varias webs diferentes:**

**Crea un archivo de configuración global:**

sudo nano /etc/apache2/conf-available/cloudflare-https.conf



**Pega esto:**

<IfModule mod_setenvif.c>
  SetEnvIf X-Forwarded-Proto https HTTPS=on
</IfModule>



**Guarda y cierra. Luego actívalo:**

sudo a2enconf cloudflare-https
sudo systemctl restart apache2



**Si quieres expandir el limite de subida de WordPress de forma gloval, edita el archivo de configuración de PHP:**


sudo nano /etc/php/[TU_VERSION_DE_PHP]/mods-available/custom.ini


**Pega esto (puedes ajustar los valores a lo que necesites):**

<IfModule mod_php.c>
  upload_max_filesize = 128M
  post_max_size = 128M
  max_execution_time = 300
  max_input_time = 300
  memory_limit = 256M
</IfModule>


**Guarda y cierra. Luego actívalo y reinicia Apache:**

sudo phpenmod custom
sudo systemctl restart apache2