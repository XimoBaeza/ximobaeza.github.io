---
layout: single
title: Academy - Hack The Box
excerpt: "Resolución de la máquina *Academy* de Hack The Box, una máquina Linux de dificultad fácil según la plataforma en la cual nos aprovechamos del parámetro rolid para crear un usuario administrador en la web, luego utilizamos un exploit para el framework *laravel* para acceder al sistema y por último aprovechamos que podemos ejecutar el comando *composer* con sudo sin contraseña para escalar privilegios y obtener una shell de root."
date: 2021-03-07
classes: wide
header:
  teaser: /assets/images/Academy-Hackthebox/academy.png
  teaser_home_page: true
categories:
  - HackTheBox
tags:
  - Web Exploiting
  - Privilege Escalation
  - Pentesting
  - Linux
---

![](/assets/images/Academy-Hackthebox/academy.png)

Resolución de la máquina *Academy* de Hack The Box, una máquina Linux de dificultad fácil según la plataforma en la cual nos aprovechamos del parámetro rolid para crear un usuario administrador en la web, luego utilizamos un exploit para el framework *laravel* para acceder al sistema y por último aprovechamos que podemos ejecutar el comando *composer* con sudo sin contraseña para escalar privilegios y obtener una shell de root.

## Escaneo de puertos

![](/assets/images/Academy-Hackthebox/nmap.png)

Puertos 22,80 y 33060 abiertos.
Empiezo por el 80. Accedo con el navegador a la ip 10.10.10.215 y me dice que no se puede abrir la página.
Veo que en el escaneo con nmap aparece el dominio academy.htb, lo añado al fichero hosts y ya me deja acceder con el nombre del dominio.

![](/assets/images/Academy-Hackthebox/web.png)

Lanzo gobuster para fuzzear los directorios y ficheros de la web.

![](/assets/images/Academy-Hackthebox/gobuster.png)

Mientras tanto me registro en la página y capturo la petición con burpsuite.
Veo que los parámetros que viajan en la petición son:
uid=ximo&password=ximo&confirm=ximo&roleid=0
Me llamala atención el parámetro roleid. Envío la petición al repeater y modifico el valor a 1.
Gobuster descubre una página que se llama admin.php, así que pruebo el usuario que he registrado con el roleid=1 y accedo como admin.
Gobuster encuentra también un fichero readme dentro del directorio academy, que dice que se está usando un framework llamado laravel.


