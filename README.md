# DOCUMENTACIÓN DE PROYECTO: COMPILACIÓN E IMPLEMENTACIÓN DE SERVIDORES NGINX Y PHP

**INSTITUCIÓN:** Tecnológico de Estudios Superiores del Oriente del Estado de México (TESOEM)
**PROFESOR:** ROMERO GONZALEZ GUSTAVO MOISES
**MATERIA:** TALLER DE SISTEMAS OPERATIVOS  
**FORMATO:** MARKDOWN (`.md`)  

---

## CARÁTULA

* **Integrantes:**
    1. VALDEZ AVENDAÑO EDUARDO
    2. DOMINGUEZ RIOS ESTEFANY MONTSERRAT
    3. LARA LOPEZ LUIS ARMANDO
* **Grupo:** 6S21
* **Fecha de entrega:** 25 DE MAYO DE 2026

---

## 1. OBJETIVOS

### 1.1 Objetivo General
Instalar, configurar y poner en marcha un entorno de servidor web compuesto por **NGINX (v1.31.x)** y **PHP (v8.4.x)** mediante la compilación de sus códigos fuente en el sistema operativo, automatizando sus servicios a través de unidades de SystemD y logrando su interconexión exclusiva por medio de sockets Unix locales.

### 1.2 Objetivos Específicos
* Compilar NGINX desde su código fuente estableciendo el prefix de instalación en el directorio `/srv/nginx` y asignando la propiedad del servicio al usuario y grupo del sistema `nginx`.
* Compilar PHP en su versión 8.4.x habilitando soporte para `php-fpm`, junto con las extensiones nativas para procesamiento de imágenes (`gd`), fechas e internacionalización (`intl`), con prefix en `/srv/nginx` y permisos compartidos `php:nginx`.
* Crear y registrar los archivos de servicio en SystemD (`nginx.service` y `php-fpm8.4.service`) para permitir el auto-arranque del servidor web durante el inicio del sistema operativo en el target `multi-user.target`.
* Configurar la comunicación FastCGI entre NGINX y PHP-FPM a través de un socket UNIX ubicado de forma dedicada en `/tmp/php84.sock`.
* Validar la correcta implementación desplegando un script de prueba (`phpinfo.php`) accesible y renderizado desde un navegador web.

---

## 2. DESARROLLO DEL PROYECTO

### Paso 2.1: Preparación del Entorno e Instalación de Dependencias
Antes de compilar, es necesario instalar las herramientas esenciales de desarrollo (`build-essential`) y las librerías necesarias para el correcto procesamiento de textos, cifrados, imágenes, fechas e internacionalización en PHP y NGINX:

```bash
sudo apt update
sudo apt install -y build-essential libpcre3-dev zlib1g-dev libssl-dev libxml2-dev libsqlite3-dev libcurl4-openssl-dev libpng-dev libjpeg-dev libicu-dev pkg-config

![Paso 2.2: Creación de Usuarios y Grupos](https://github.com/user-attachments/assets/dd373f94-4b0b-4e0d-b06c-df4a1df6b968)



### Paso 2.2: Creación de Usuarios y Grupos de Sistemas
Se procedió a estructurar los usuarios encargados de la ejecución de los servicios, restringiendo el acceso a la Shell por motivos estrictos de seguridad (nologin):

# Crear grupo y usuario de sistema para NGINX
sudo useradd -r -M -s /usr/sbin/nologin nginx

# Crear usuario de sistema para PHP asignándolo simultáneamente al grupo nginx
sudo useradd -r -M -s /usr/sbin/nologin -g nginx php

CAPTURAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

### Paso 2.3: Descarga y Compilación de NGINX (v1.31.x)
Se descargó el código fuente de la rama de desarrollo 1.31.x de NGINX, se desempaquetó y se configuró con las directivas del prefix de destino, usuario y grupo del sistema:

# Descarga y descompresión del fuente
wget [http://nginx.org/download/nginx-1.31.0.tar.gz](http://nginx.org/download/nginx-1.31.0.tar.gz)
tar -xzvf nginx-1.31.0.tar.gz
cd nginx-1.31.0

# Configuración con prefix de instalación solicitado
./configure --prefix=/srv/nginx --user=nginx --group=nginx --with-http_ssl_module

# Compilación e instalación en el sistema
make
sudo make install

CAPTURAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

### Paso 2.4: Registro del Servicio SystemD para NGINX
Para cumplir con la automatización del arranque, se creó y registró un Slice de Servicio de SystemD mediante el archivo /etc/systemd/system/nginx.service:

[Unit]
Description=A high performance web server compiled from source
After=network.target

[Service]
Type=forking
PIDFile=/srv/nginx/logs/nginx.pid
ExecStartPre=/srv/nginx/sbin/nginx -t
ExecStart=/srv/nginx/sbin/nginx
ExecReload=/srv/nginx/sbin/nginx -s reload
ExecStop=/srv/nginx/sbin/nginx -s stop
PrivateTmp=true

[Install]
WantedBy=multi-user.target

Posteriormente, se recargó el demonio de SystemD, se habilitó para arrancar en el inicio del SO (multi-user.target) y se inició el proceso:

sudo systemctl daemon-reload
sudo systemctl enable nginx.service
sudo systemctl start nginx.service

CAPTURAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

### Paso 2.5: Descarga y Compilación de PHP (v8.4.x) con Soporte Extendido
Se descargó la versión más reciente de la rama PHP 8.4.x, configurando explícitamente el módulo php-fpm, el prefix común asignado, los usuarios correspondientes y las extensiones obligatorias para imágenes (gd), internacionalización (intl) y fechas:

# Descarga y descompresión de PHP
wget [https://www.php.net/distributions/php-8.4.0.tar.gz](https://www.php.net/distributions/php-8.4.0.tar.gz)
tar -xzvf php-8.4.0.tar.gz
cd php-8.4.0

# Configuración del compilador habilitando FPM, GD e INTL
./configure --prefix=/srv/nginx \
            --enable-fpm \
            --with-fpm-user=php \
            --with-fpm-group=nginx \
            --enable-intl \
            --with-external-gd \
            --enable-mbstring

# Compilación de binarios e instalación
make
sudo make install

CAPTURAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

### Paso 2.6: Configuración del Socket UNIX de PHP-FPM
Para enlazar PHP y NGINX, se configuró el archivo del pool de conexiones (ubicado por defecto de la compilación en /srv/nginx/etc/php-fpm.d/www.conf o renombrado desde www.conf.default). Se modificaron las directivas para forzar el uso del Socket UNIX con los permisos asignados:

; Configuración del socket solicitado en la ruta especificada
listen = /tmp/php84.sock

; Asignación de permisos al grupo y usuario dueños del servicio
listen.owner = php
listen.group = nginx
listen.mode = 0660

CAPTURAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

### Paso 2.7: Registro del Servicio SystemD para PHP-FPM
Se generó el archivo de control para la automatización de PHP-FPM en la ruta /etc/systemd/system/php-fpm8.4.service:

[Unit]
Description=The PHP 8.4 FastCGI Process Manager compiled from source
After=network.target

[Service]
Type=simple
ExecStart=/srv/nginx/sbin/php-fpm --nodaemonize
ExecReload=/bin/kill -USR2 $MAINPID
PrivateTmp=false

[Install]
WantedBy=multi-user.target

Habilitación del servicio para su auto-arranque continuo:

sudo systemctl daemon-reload
sudo systemctl enable php-fpm8.4.service
sudo systemctl start php-fpm8.4.service

CAPTURAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

### Paso 2.8: Configuración de la Comunicación FastCGI en NGINX
Se editó el bloque de servidores del archivo principal de NGINX localizado en /srv/nginx/conf/nginx.conf para indicarle que debe redirigir las peticiones .php hacia el socket UNIX creado por PHP-FPM:

worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       80;
        server_name  localhost;
        root         /srv/nginx/html;
        index        index.php index.html index.htm;

        location / {
            try_files $uri $uri/ =404;
        }

        # Enlace FastCGI hacia el Socket UNIX /tmp/php84.sock
        location ~ \.php$ {
            include        fastcgi_params;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            fastcgi_pass   unix:/tmp/php84.sock;
        }
    }
}

Verificación de la sintaxis y reinicio de los servicios web:

sudo /srv/nginx/sbin/nginx -t
sudo systemctl restart nginx

CAPTURAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

## 3. COMPROBACIÓN DE FUNCIONAMIENTO
Para comprobar la correcta interconexión operativa entre ambos servicios de red compilados, se creó un script ejecutable de diagnóstico en la raíz de documentos establecida /srv/nginx/html/phpinfo.php:

sudo nano /srv/nginx/html/phpinfo.php

Contenido embebido:

<?php
phpinfo();
?>

## Resultado Exitoso en el Navegador Web:
Al consultar desde el navegador mediante la dirección IP local del servidor (http://<IP_LOCAL>/phpinfo.php), se despliega exitosamente la interfaz de diagnóstico del intérprete, donde se certifica de forma visual la versión activa de PHP 8.4.x, el servidor web NGINX 1.31.x, y el correcto procesamiento de los módulos internos de fechas, imágenes (GD) e internacionalización (intl).

CAPTURAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

## 4. CONCLUSIONES
La compilación nativa desde código fuente permite prescindir de paquetes innecesarios para el sistema, aislando las rutas operativas en un prefijo ordenado (/srv/nginx), garantizando el máximo rendimiento del entorno en producción.

La implementación de comunicación local basada en sockets UNIX resulta sustancialmente más eficiente que la alternativa de Sockets TCP (localhost/127.0.0.1), ya que elimina el retardo (overhead) derivado de procesar paquetes a través de los protocolos de la pila de red local Loopback.

La integración directa bajo la tutela de SystemD en Linux asegura la robustez requerida para un entorno de servidor, controlando el ciclo de ejecución de los demonios web y garantizando su disponibilidad continua al activarse dinámicamente durante el arranque del sistema.

## 5. BIBLIOGRAFÍA (Formato APA)
The NGINX Project. (2026). NGINX Open Source Documentation. Recuperado de https://nginx.org/en/docs/

The PHP Group. (2026). PHP: Hypertext Preprocessor Manual - FastCGI Process Manager (FPM). Recuperado de https://www.php.net/manual/en/install.fpm.php

The systemd Project. (2025). systemd.service — Service unit configuration. Recuperado de https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html
