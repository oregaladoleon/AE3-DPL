# AE3-DPL
Galería con procesado de imágenes, módulos y HTTPS

> Autor: Óscar Regalado León  
> Fecha: Octubre 2026 

## Objetivo
Desplegar una galería Web en images.alumno.local que procese 20 imágenes on the fly con el módulo image_filter de nginx, protegida con autenticación, servida solo por HTTPS y con la actividad monitorizada mediante goaccess.

~~~mermaid
graph LR
    A["Navegador: galería<br/>(index.html + app.js)"] -->|actualiza src de 20 img| B["/img/imageNN.jpg?<br/>size=N&q=Q&sharpen=P"]
    B -->|HTTPS + auth_basic| C["nginx images.alumno.local"]
    C -->|image_filter resize + sharpen| D["/var/www/<br/>images.alumno.local/img/<br/>*.jpg"]
    C --> E["images-alumno.access.log"]
    E --> F["goaccess<br/>(JSON/HTML)"]
~~~
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
Aplicamos el módulo de **ImageMagick** para generar 20 imágenes dentro del directorio /var/www/images.alumno.local/img:
~~~bash
docker exec dpl-lab mkdir -p /var/www/images.alumno.local/img

docker exec dpl-lab bash -c 'for i in $(seq -w 1 20); do \
  convert -size 400x400 gradient:"#2f6fed"-"#8bc34a" \
    -fill white -pointsize 64 -gravity center -annotate 0 "IMG $i" \
    /var/www/images.alumno.local/img/image$i.jpg; done'

docker exec dpl-lab identify /var/www/images.alumno.local/img/image01.jpg   # 400x400
~~~

Las 20 imágenes de prueba no están incluidas en el control de versiones (están excluidas mediante el .gitignore) por tratarse de **ficheros binarios derivados**. En su lugar, el repositorio incluye el comando exacto de ImageMagick necesario para generarlas de forma automatizada en el entorno de laboratorio, garantizando la reproducibilidad del proyecto sin aumentar inecesariamente el tamaño del repositorio de Git.

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
Generamos el directorio dentro del contenedor donde se alojarán los ficheros del sitio web y copiamos el archivo de configuración desde el directorio /home/alumno/dpl/ del contenedor a /etc/nginx/sites-available:
~~~bash
docker exec dpl-lab mkdir -p /var/www/images.alumno.local

docker cp /home/alumno/dpl/AE3/nginx/images.alumno.local.conf /etc/nginx/sites-available/  
~~~
---
Para generar el certificado SSL con SAN procedemos:
~~~bash
docker exec dpl-lab mkdir -p /etc/ssl/dpl

docker exec dpl-lab openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
     -keyout /etc/ssl/dpl/images.alumno.local.key \
     -out /etc/ssl/dpl/images.alumno.local.crt \
     -subj "/CN=images.alumno.local" \
     -addext "subjectAltName=DNS:images.alumno.local,DNS:www.images.alumno.local"
~~~
Además, copiamos la parte pública del certificado en el repositorio local.
~~~bash
docker cp dpl-lab:/etc/ssl/dpl/images.alumno.local.crt certs/images.alumno.local.crt
~~~

Es necesario generar unas credenciales **htpasswd** de usuario y contraseña, para ello:
~~~bash
docker exec dpl-lab bash -c 'echo "alumno:$(openssl passwd -apr1 CAMBIAR_ESTO)" > /etc/nginx/htpasswd-images'

docker exec dpl-lab chown root:www-data /etc/nginx/htpasswd-images

docker exec dpl-lab chmod 640 /etc/nginx/htpasswd-images
~~~
---
Para activar el sitio web, creamos el enalce simbólico entre los directorios de /etc/nginx/sites-enabled y /etc/nginx/sites-available:
~~~bash
docker exec dpl-lab ln -sf /etc/nginx/sites-available/images.alumno.local.conf /etc/nginx/sites-enabled/

docker exec dpl-lab nginx -t

docker exec dpl-lab nginx -s reload
~~~
Para la creación de los ficheros del sitio web se cogen los propuestos en la práctica guiada.
~~~bash
nano ~/dpl/AE3/web/index.html

nano ~/dpl/AE3/web/estilos.css

nano ~/dpl/AE3/web/app.js
~~~
Estos ficheros se copian en la ruta del sitio web dentro del contenedor:
~~~bash
docker cp web/. dpl-lab:/var/www/images.alumno.local/
~~~
## La aplicación

**image_filter** solo aplica transformaciones como resize, crop, rotate, sharpen y calidad; no soporta bordes, colores ni desenfoque. Por eso la galería aplica el borde y su color con CSS y el desenfoque con filter: blur(...), mientras que el tamaño y el enfoque viajan en la URL hasta nginx. Un módulo de terceros como ngx_small_light sí permitiría bordes y desenfoque en el servidor, pero requiere compilarlo contra la versión exacta de nginx: es una ampliación opcional, no probada en este curso.

## Comprobación
Los resultados obtenidos de las comprobaciones han sido:
~~~bash
curl -k -I https://localhost/ -H "Host: images.alumno.local"
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Date: Mon, 19 Oct 2026 18:46:56 GMT
Content-Type: text/html
Content-Length: 1350
Last-Modified: Mon, 19 Oct 2026 15:27:29 GMT
Connection: keep-alive
ETag: "6ad636e1-546"
Accept-Ranges: bytes
~~~

~~~bash
curl -k -I "https://localhost/img/image01.jpg" -H "Host: images.alumno.local"
HTTP/1.1 401 Unauthorized
Server: nginx/1.24.0 (Ubuntu)
Date: Mon, 19 Oct 2026 18:50:25 GMT
Content-Type: text/html
Content-Length: 188
Connection: keep-alive
WWW-Authenticate: Basic realm="Acceso restringido - Galeria AE3"
~~~

~~~bash
curl -k -u alumno:CAMBIAR_ESTO -o /tmp/mini.jpg \
    "https://localhost/img/image01.jpg?size=150&q=60&sharpen=30" -H "Host: images.alumno.local"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2232  100  2232    0     0  25368      0 --:--:-- --:--:-- --:--:-- 25655

identify /tmp/mini.jpg
/tmp/mini.jpg JPEG 150x150 150x150+0+0 8-bit sRGB 2232B 0.000u 0:00.000
~~~

~~~bash
curl -I http://localhost/ -H "Host: images.alumno.local"
HTTP/1.1 301 Moved Permanently
Server: nginx/1.24.0 (Ubuntu)
Date: Mon, 19 Oct 2026 18:54:52 GMT
Content-Type: text/html
Content-Length: 178
Connection: keep-alive
Location: https://images.alumno.local/
~~~

~~~bash
goaccess /var/log/nginx/images-alumno.access.log \
    --log-format=COMBINED -o /home/alumno/dpl/informe-images.json
[PARSING /var/log/nginx/images-alumno.access.log] {0} @ {0/s}
Cleaning up resources...

goaccess /var/log/nginx/images-alumno.access.log \
    --log-format=COMBINED -o /home/alumno/dpl/informe-images.html
[PARSING /var/log/nginx/images-alumno.access.log] {0} @ {0/s}
Cleaning up resources...

~~~
## Problemas encontrados
Al aplicar el siguiente comando con credenciales incorrectas:
~~~bash
docker exec -it dpl-lab curl -k -u alumno:CAMBIAR_ESTO -o /tmp/mini.jpg \
    "https://localhost/img/image01.jpg?size=150&q=60&sharpen=30" \
    -H "Host: images.alumno.local"
~~~
Se obtuvo un error 415 Unsupported Media Type, en vez del esperado 401 Unauthorized.

Tras investigar, se detectó que en el fichero de configuración ~/dpl/AE3/nginx/images.alumno.local.conf no estaba especificada la directriz **error_page 401 @sin_autorizacion;**. Se especificó en el documento y se solucionó el problema.

## Enlace al repositorio remoto 
Enlace público: https://github.com/oregaladoleon/AE3-DPL.git