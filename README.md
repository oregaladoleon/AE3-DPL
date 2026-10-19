# AE3-DPL
Galería con procesado de imágenes, módulos y HTTPS

> Autor: Óscar Regalado León  
> Fecha: Octubre 2026 

## Objetivo
Desplegar una galería Web en images.alumno.local que procese 20 imágenes on the fly con el módulo image_filter de nginx, protegida con autenticación, servida solo por HTTPS y con la actividad monitorizada mediante goaccess.

!!!Incluir diagrama mermeid

## Índice

- [Instalación](#Instalación)
- [Generación de las imágenes](#Generación-de-las-imágenes)
- [Configuración](#configuración)
- [La aplicación](#La-aplicación)
- [Comprobación](#comprobación)
- [Problemas encontrados](#problemas-encontrados)
- [Enlace al repositorio remoto](#Enlace-al-repositorio-remoto)


## Instalación
Inicializamos el repositorio en la web de **Github** y clonamos dentro de la carpeta de trabajo ~/dpl/AE3, también levantamos el contenedor docker dpl-lab y creamos el fichero **comprobaciones.txt**:
~~~bash
git clone git@github.com:oregaladoleon/AE3-DPL.git AE3

touch .gitignore  #Añadimos la configuración adecuada para esta práctica.

touch comprobaciones.txt

sudo docker ps -a

sudo docker start dpl-lab
~~~
El fichero **.gitignore** le añadimos:
~~~
# Created by https://www.toptal.com/developers/gitignore/api/linux,visualstudiocode
# Edit at https://www.toptal.com/developers/gitignore/api/linux,visualstudiocode

### Linux ###
*~
# temporary files which can be created if a process still has an open file of a deleted file
.fuse_hidden*
# System volume information
.directory
# Trash folder which might appear on any partition or disk
.Trash-*
# .nfs files
.nfs*

### VisualStudioCode ###
.vscode/*
!.vscode/settings.json
!.vscode/tasks.json
!.vscode/launch.json
!.vscode/extensions.json
!.vscode/*.code-snippets

# Local History for Visual Studio Code
.history/

# Built Local Extension Files
*.vsix

### Nginx ###
*.log
logs/
nginx.pid
nginx.app.pid

### AE3 - Reglas de seguridad y temporales (OBLIGATORIAS) ###
# Claves privadas SSL 
*.key
*.pem
certs/*.key

# Ficheros de contraseñas de autenticación básica
htpasswd*
.htpasswd

# Salidas de GoAccess e informes temporales
/tmp/*.json

# Imágenes de prueba generadas con ImageMagick 
web/img/
img/
~~~
Para la creación del árbol del proyecto procedemos de la siguiente manera:
~~~bash
mkdir -p nginx web certs
~~~

Instalamos las dependencias en el contenedor, con los módulos **nginx**, **ImageMagick**, **GoAccess**:
~~~bash
docker exec -it dpl-lab apt-get update

docker exec -it dpl-lab apt-get install -y libnginx-mod-http-image-filter imagemagick goaccess
~~~
Para verificar que el módulo **image_filter** está en la compilación de **nginx**, aplicamos:
~~~bash
docker exec dpl-lab nginx -V 2>&1 | grep image_filter

docker exec dpl-lab nginx -t

ls -l /etc/nginx/modules-enabled/  #Aquí podemos observar el enlace simbólico de activación.
~~~

## Generación de las imágenes 

## Configuración

Creamos el fichero de configuración en **~/dpl/AE3/nginx**, el fichero es **images.alumno.local.conf**:
~~~bash
nano ~/dpl/AE3/nginx/images.alumno.local.conf
~~~
Añadimos la siguiente configuración:
~~~
 Mapas con listas blancas para validar los parámetros de tamaño y enfoque
map $arg_size $img_size {
    default 300;
    150     150;
    300     300;
    600     600;
}

map $arg_q $img_quality {
    default 80;
    60      60;
    80      80;
    100     100;
}

map $arg_sharpen $img_sharpen {
    default 0;
    0       0;
    30      30;
    60      60;
    100     100;
}

# Redirección global de HTTP (puerto 80) a HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name images.alumno.local www.images.alumno.local;
    return 301 https://images.alumno.local$request_uri;
}

# Servidor principal HTTPS para images.alumno.local
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name images.alumno.local;

    root /var/www/images.alumno.local;
    index index.html;

    ssl_certificate /etc/ssl/dpl/images.alumno.local.crt;
    ssl_certificate_key /etc/ssl/dpl/images.alumno.local.key;

    # Localización general
    location / {
        try_files $uri $uri/ =404;
    }

    # Localización protegida y exclusiva con image_filter y auth_basic
    location /img {
        auth_basic "Acceso restringido - Galeria AE3";
        auth_basic_user_file /etc/nginx/htpasswd-images;
        # [CRÍTICO] Evita que un error 401 (Unauthorized) devuelva un HTML 
        # que image_filter rechace convirtiéndolo en un error 415.
        error_page 401 @sin_autorizacion;

        image_filter resize $img_size $img_size;
        image_filter_sharpen $img_sharpen;
        image_filter_buffer 10M;
       
    }

    location @sin_autorizacion {
        return 401;
    }

    access_log /var/log/nginx/images-alumno.access.log combined;
    error_log  /var/log/nginx/images-alumno.error.log;
}
~~~

## La aplicación

## Problemas encontrados

