# AE3-DPL
Galería con procesado de imágenes, módulos y HTTPS

> Autor: Óscar Regalado León  
> Fecha: Octubre 2026 

## Objetivo
Desplegar una galería Web en images.alumno.local que procese 20 imágenes on the fly con el módulo image_filter de nginx, protegida con autenticación, servida solo por HTTPS y con la actividad monitorizada mediante goaccess.

## Índice

- [Instalación](#Instalación)
- [Generación de las imágenes](#Generación-de-las-imágenes)
- [Configuración](#configuración)
- [La aplicación](#La-aplicación)
- [Comprobación](#comprobación)
- [Problemas encontrados](#problemas-encontrados)
- [Enlace al repositorio remoto](#Enlace-al-repositorio-remoto)


## Instalación
Inicializamos el repositorio dentro de la carpeta de trabajo ~/dpl/reto-https-ute3, también levantamos el contenedor docker dpl-lab2:
~~~bash
mkdir -p ~/dpl/reto-https-ute3
git init
touch .gitignore  #Añadimos la configuración adecuada para esta práctica.
touch README.md 
touch comandos.txt
touch config.txt
sudo docker ps -a
sudo docker start dpl-lab2
~~~
El fichero **.gitignore** le añadimos:
~~~
*.log
logs/
htpasswd*
*.key
*.pem
*~
*.swp
.DS_Store
~~~

## Generación de las imágenes 

## Configuración

## La aplicación

## Problemas encontrados

