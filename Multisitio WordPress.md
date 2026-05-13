# WordPress Multisite con Apache2 y Cloudflare Tunnel en Ubuntu Server

> Guía completa para configurar WordPress Multisite con subdominios en una VM Ubuntu Server, usando Apache2 como servidor web y Cloudflare Tunnel para exposición a internet sin abrir puertos.

---

## 📋 Índice

1. [Requisitos previos](#requisitos-previos)
2. [Arquitectura final](#arquitectura-final)
3. [Mover WordPress a su carpeta definitiva](#1-mover-wordpress-a-su-carpeta-definitiva)
4. [Configurar los VirtualHosts de Apache2](#2-configurar-los-virtualhost-de-apache2)
5. [Habilitar WordPress Multisite](#3-habilitar-wordpress-multisite)
6. [Instalar la red desde el panel de WordPress](#4-instalar-la-red-desde-el-panel-de-wordpress)
7. [Aplicar los cambios en wp-config.php y .htaccess](#5-aplicar-los-cambios-en-wp-configphp-y-htaccess)
8. [Crear el sitio del subdominio](#6-crear-el-sitio-del-subdominio)
9. [Configurar Cloudflare Tunnel](#7-configurar-cloudflare-tunnel)
10. [Reactivar plugins](#8-reactivar-plugins)
11. [Verificación final](#9-verificación-final)

---

## Requisitos previos

- Ubuntu Server (esta guía usa 24.04)
- Apache2 instalado y funcionando
- WordPress descargado e instalado
- MySQL/MariaDB con la base de datos de WordPress creada
- Cuenta en Cloudflare con tu dominio apuntando a Cloudflare
- Cloudflare Tunnel (`cloudflared`) instalado y corriendo en la VM
- Dominio propio (en esta guía usamos `justecno.com` como ejemplo)

---

## Arquitectura final

```
Internet
   ↓
Cloudflare (gestiona justecno.com y cv.justecno.com)
   ↓
Cloudflare Tunnel (un solo túnel, sin abrir puertos)
   ↓
Ubuntu Server → Apache2 en puerto 80/443
                    ↓
         VirtualHost Apache decide por dominio:
         ┌─────────────────────────────────────┐
         │ justecno.com    → WordPress Sitio 1  │
         │ cv.justecno.com → WordPress Sitio 2  │
         └─────────────────────────────────────┘
```

**WordPress Multisite** gestiona ambos sitios desde una sola instalación:
- Una sola base de datos
- Un solo panel de administración
- Header/footer compartido automáticamente entre sitios
- Usuarios compartidos entre los dos sitios

---

## 1. Mover WordPress a su carpeta definitiva

Por defecto WordPress suele instalarse en `/var/www/html/WordPress`. Lo movemos a una ruta más organizada.

**Primero, localiza dónde está instalado:**
```bash
find /var/www -name "wp-config.php" 2>/dev/null
```

En nuestro caso estaba en `/var/www/html/WordPress`. Lo movemos:

```bash
sudo mv /var/www/html/WordPress /var/www/justecno.com
```

> ⚠️ Reemplaza `justecno.com` con tu dominio real en todos los comandos de esta guía.

---

## 2. Configurar los VirtualHost de Apache2

### Renombrar el archivo de configuración existente

```bash
sudo mv /etc/apache2/sites-available/WordPress.conf /etc/apache2/sites-available/justecno.com.conf
```

### Editar el VirtualHost principal (HTTP)

```bash
sudo nano /etc/apache2/sites-available/justecno.com.conf
```

Contenido completo:

```apache
<VirtualHost *:80>
    ServerName justecno.com
    ServerAlias www.justecno.com
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/justecno.com

    <Directory /var/www/justecno.com>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

### Configurar HTTPS interno (opcional pero recomendado)

Edita el archivo SSL por defecto:

```bash
sudo nano /etc/apache2/sites-available/default-ssl.conf
```

Agrega `ServerName` y actualiza `DocumentRoot`:

```apache
<VirtualHost *:443>
    ServerName justecno.com
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/justecno.com

    SSLEngine on
    SSLCertificateFile      /etc/ssl/certs/ssl-cert-snakeoil.pem
    SSLCertificateKeyFile   /etc/ssl/private/ssl-cert-snakeoil.key

    <FilesMatch "\.(?:cgi|shtml|phtml|php)$">
        SSLOptions +StdEnvVars
    </FilesMatch>
    <Directory /usr/lib/cgi-bin>
        SSLOptions +StdEnvVars
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

### Activar módulos y sitios necesarios

```bash
sudo a2enmod rewrite
sudo a2enmod ssl
sudo a2ensite justecno.com.conf
sudo a2ensite default-ssl.conf
sudo systemctl reload apache2
```

### Verificar que WordPress responde

```bash
curl -I http://localhost
```

Debes ver `HTTP/1.1 200 OK` en la respuesta.

---

## 3. Habilitar WordPress Multisite

Edita el archivo de configuración de WordPress:

```bash
sudo nano /var/www/justecno.com/wp-config.php
```

Busca la sección de valores personalizados (entre los comentarios `/* Add any custom values...*/` y `/* That's all, stop editing! */`) y agrega:

```php
define('WP_ALLOW_MULTISITE', true);
```

Guarda el archivo.

---

## 4. Instalar la red desde el panel de WordPress

1. Ve a `tudominio.com/wp-admin`
2. En el menú lateral: **Herramientas → Configuración de la red**

> ⚠️ **Importante:** WordPress pedirá que desactives todos los plugins antes de continuar. Ve a **Plugins → Plugins instalados**, selecciona todos y elige **Desactivar** en acciones en lote.

3. Regresa a **Herramientas → Configuración de la red**
4. Selecciona **Subdominios** (para usar `cv.tudominio.com`)
5. Rellena los campos:
   - **Título de la red:** El nombre de tu red (ej: `MiSitio`)
   - **Correo electrónico:** Tu email de administrador
6. Haz clic en **Instalar**

WordPress generará dos bloques de código que debes copiar en el siguiente paso.

> **Nota:** Puede aparecer una advertencia sobre DNS. Es normal en este punto, ignórala.

---

## 5. Aplicar los cambios en wp-config.php y .htaccess

WordPress te mostrará exactamente qué agregar. Copia los bloques que te da y aplícalos así:

### wp-config.php

```bash
sudo nano /var/www/justecno.com/wp-config.php
```

Agrega **justo encima** de `/* That's all, stop editing! */`:

```php
define( 'MULTISITE', true );
define( 'SUBDOMAIN_INSTALL', true );
define( 'DOMAIN_CURRENT_SITE', 'tudominio.com' );
define( 'PATH_CURRENT_SITE', '/' );
define( 'SITE_ID_CURRENT_SITE', 1 );
define( 'BLOG_ID_CURRENT_SITE', 1 );
```

> Los valores exactos los genera WordPress automáticamente. Copia los que te muestra en pantalla.

### .htaccess

```bash
sudo nano /var/www/justecno.com/.htaccess
```

Reemplaza **todo** el contenido con:

```apache
RewriteEngine On
RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
RewriteBase /
RewriteRule ^index\.php$ - [L]

# add a trailing slash to /wp-admin
RewriteRule ^wp-admin$ wp-admin/ [R=301,L]

RewriteCond %{REQUEST_FILENAME} -f [OR]
RewriteCond %{REQUEST_FILENAME} -d
RewriteRule ^ - [L]
RewriteRule ^(wp-(content|admin|includes).*) $1 [L]
RewriteRule ^(.*\.php)$ $1 [L]
RewriteRule . index.php [L]
```

### Recargar Apache

```bash
sudo systemctl reload apache2
```

Luego haz clic en **Acceder** en la página de WordPress para volver a iniciar sesión.

---

## 6. Crear el sitio del subdominio

1. En la barra negra superior: **Mis sitios → Administrador de la red**
2. Menú lateral: **Sitios → Agregar nuevo**
3. Rellena los campos:
   - **Dirección del sitio (URL):** `cv` (WordPress agrega `.tudominio.com` automáticamente)
   - **Título del sitio:** El nombre que quieras (ej: `Mi CV`)
   - **Idioma:** El que prefieras
   - **Correo del administrador:** Tu email
4. Haz clic en **Añadir sitio**

El sitio `cv.tudominio.com` quedará creado y accesible desde el panel.

> **¿Necesito un VirtualHost separado para el subdominio?**
> No. WordPress Multisite con subdominios reutiliza el mismo VirtualHost del sitio principal. Apache lee el header `Host` de cada petición y WordPress decide qué sitio mostrar.

---

## 7. Configurar Cloudflare Tunnel

Con Cloudflare Tunnel **no necesitas abrir puertos** en tu router ni en tu VM. El túnel actúa como canal seguro entre Cloudflare e internet y tu servidor local.

### Por qué funciona con un solo túnel para dos dominios

```
cv.justecno.com  ──→  Cloudflare  ──→  Tunnel  ──→  localhost:80
justecno.com     ──→  Cloudflare  ──→  Tunnel  ──→  localhost:80
                                              ↓
                                    Apache lee header Host:
                                    y enruta al sitio correcto
```

### Agregar el subdominio al túnel existente

1. Ve a [Cloudflare Zero Trust](https://one.dash.cloudflare.com)
2. Navega a **Networks → Tunnels**
3. Haz clic en tu túnel → **Edit → Public Hostname → Add a public hostname**
4. Configura:

| Campo | Valor |
|---|---|
| Subdomain | `cv` |
| Domain | `tudominio.com` |
| Service Type | `HTTP` |
| URL | `localhost:80` |

5. Guarda los cambios.

Cloudflare creará automáticamente el registro DNS para `cv.tudominio.com`.

> **Sobre SSL:** No necesitas Certbot ni certificados en Apache para el tráfico externo. Cloudflare gestiona HTTPS entre el usuario e internet. El tráfico interno VM↔Cloudflare va cifrado por el túnel.

---

## 8. Reactivar plugins

Ahora que Multisite está funcionando, reactiva tus plugins. En Multisite hay dos tipos de activación:

| Tipo | Dónde activar | Efecto |
|---|---|---|
| **Por red** | Administrador de la red → Plugins | Disponible en TODOS los sitios |
| **Por sitio** | Panel de cada sitio → Plugins | Solo en ese sitio |

**Recomendación:**
- Plugins de seguridad, SEO, caché → actívalos **por red**
- Plugins específicos de un sitio → actívalos **solo en ese sitio**

Para activar por red:
1. **Mis sitios → Administrador de la red → Plugins**
2. Clic en **Activar en la red** bajo cada plugin

---

## 9. Verificación final

Comprueba que todo funciona:

```bash
# Verificar que Apache responde correctamente
curl -I http://localhost
curl -I http://localhost -H "Host: cv.tudominio.com"

# Verificar configuración de Apache
sudo apache2ctl configtest

# Ver sitios activos
ls /etc/apache2/sites-enabled/
```

Desde el navegador:
- `https://tudominio.com` → Sitio principal ✅
- `https://cv.tudominio.com` → Sitio del subdominio ✅
- `https://tudominio.com/wp-admin` → Panel del sitio principal ✅
- `https://cv.tudominio.com/wp-admin` → Panel del subdominio ✅
- `https://tudominio.com/wp-admin/network/` → Administrador de la red ✅

---

## 📁 Estructura de archivos resultante

```
/var/www/
└── justecno.com/          ← Instalación única de WordPress
    ├── wp-config.php       ← Contiene configuración Multisite
    ├── .htaccess           ← Reglas de reescritura Multisite
    ├── wp-content/
    │   ├── themes/
    │   ├── plugins/
    │   └── uploads/
    │       ├── sites/
    │       │   └── 2/     ← Archivos del sitio cv.tudominio.com
    │       └── ...        ← Archivos del sitio principal
    └── ...

/etc/apache2/sites-available/
├── 000-default.conf        ← VirtualHost HTTP por defecto (no modificar)
├── default-ssl.conf        ← VirtualHost HTTPS interno
└── justecno.com.conf       ← VirtualHost principal (sirve ambos sitios)
```

---

## ❓ Preguntas frecuentes

**¿Por qué no necesito un `.conf` separado para el subdominio?**
WordPress Multisite intercepta todas las peticiones a través del `.htaccess` y decide qué sitio mostrar según el dominio. Apache solo necesita conocer el sitio principal.

**¿Los usuarios se comparten entre los dos sitios?**
Sí. En Multisite hay una sola tabla de usuarios en la base de datos. Un usuario registrado en `tudominio.com` puede tener acceso a `cv.tudominio.com` si le asignas un rol en ese sitio.

**¿Puedo tener más de dos sitios?**
Sí, puedes crear tantos subdominios como necesites desde **Administrador de la red → Sitios → Agregar nuevo**.

**¿Qué pasa si desactivo Multisite?**
No se recomienda una vez activado. Requiere reversión manual de `wp-config.php`, `.htaccess` y la base de datos.

**¿Puedo usar un dominio completamente diferente en lugar de un subdominio?**
Sí, con el plugin **WordPress MU Domain Mapping** puedes asignar cualquier dominio a cualquier sitio de la red.

---

## 🔗 Referencias

- [WordPress Multisite — Documentación oficial](https://developer.wordpress.org/advanced-administration/multisite/)
- [Apache VirtualHost — Documentación oficial](https://httpd.apache.org/docs/2.4/vhosts/)
- [Cloudflare Tunnel — Documentación oficial](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)

---

*Guía elaborada sobre Ubuntu Server 24.04, Apache 2.4.58, WordPress 6.9.4*