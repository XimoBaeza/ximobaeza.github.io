---
layout: single
title: Passage - Hack The Box
excerpt: "Resolución de la máquina *Passage* de Hack The Box, una máquina Linux de dificultad media según la plataforma en la cual nos aprovechamos de un exploit para el gestor de noticias *CuteNews* para obtener la intrusión inicial. Después encontraremos el hash del usuario paul y lo crackearemos para obtener su contraseña y migrar a su centa de usuario. Veremos como éste usuario tiene la clave privada ssh del usuario navdav en su directorio home, la utilizaremos y migraremos a su cuenta de usuario, y finalmente nos aprovecharemos de una vulnerabilidad que nos permite crear ficheros con permisos de root para poner nuestra clave pública de ssh en las claves autorizadas del usuario root para conectarnos por ssh sin contraseña con la cuenta de root."
date: 2021-03-08
classes: wide
header:
  teaser: /assets/images/Passage-Hackthebox/passage.png
  teaser_home_page: true
categories:
  - HackTheBox
tags:
  - Web Exploiting
  - Privilege Escalation
  - Pentesting
  - Linux
---

![](/assets/images/Passage-Hackthebox/passage.png)

Resolución de la máquina *Passage* de Hack The Box, una máquina Linux de dificultad media según la plataforma en la cual nos aprovechamos de un exploit para el gestor de noticias *CuteNews* para obtener la intrusión inicial. Después encontraremos el hash del usuario paul y lo crackearemos para obtener su contraseña y migrar a su centa de usuario. Veremos como éste usuario tiene la clave privada ssh del usuario navdav en su directorio home, la utilizaremos y migraremos a su cuenta de usuario, y finalmente nos aprovecharemos de una vulnerabilidad que nos permite crear ficheros con permisos de root para poner nuestra clave pública de ssh en las claves autorizadas del usuario root para conectarnos por ssh sin contraseña con la cuenta de root.

## Escaneo de puertos

![](/assets/images/Passage-Hackthebox/nmap.png)

Puertos 22 y 80 abiertos.

Empiezo mirando la web en el puerto 80.

![](/assets/images/Passage-Hackthebox/web.png)

No hay robots.txt y en el código fuente tampoco hay nada interesante.

Decido fuzzear con ffuf pero me doy cuenta de que a los pocos segundos de empezar a buscar directorios se corta el acceso a la web, como si me hubieran baneado la ip.

## Intrusión inicial

En el pie de página veo “Powered by CuteNews” así que pruebo con searchsploit a ver si encuentra algo relacionado.

Y bingo! Encuentro ésto CuteNews 2.1.2 - Remote Code Execution

Lo descargo con searchsploit -m php/webapps/48800.py, lo ejecuto y me da diréctamente ejecución de comandos.

![](/assets/images/Passage-Hackthebox/cutenews.png)

Lo siguiente es ponerme en escucha en el puerto 9001 con rlwrap nc -lvnp 9001 y ejecutar una reverse shell en bash para recibir la shell.

![](/assets/images/Passage-Hackthebox/shell.png)

Convierto la shell en una shell totalmente interactiva.
Con which python compruebo de tiene python instalado, y luego como siempre:
```
python -c ‘import pty;pty.spawn(“/bin/bash”)’
ctrl + z
stty raw -echo
fg
export TERM=xterm
```

![](/assets/images/Passage-Hackthebox/interactiva.png)


## Escalada al usuario paul

En el directorio home veo que hay dos usuarios, paul y nadav, pero no puedo listar lo que hay dentro.
Empiezo a enumerar el directorio var/www/html/CuteNews y encuentro un fichero que tiene unos hashes en base64.

El fichero es var/www/html/CuteNews/cdata/users/lines
Y contiene lo siguiente:

![](/assets/images/Passage-Hackthebox/users.png)

Mirando con `echo “texto” | base64 -d` en cada una de las cadenas de texto veo que hay algunas que tienen un hash de usuario.

```
a:1:{s:4:"name";a:1:{s:10:"paul-coles";a:9:

{s:2:"id";s:10:"1592483236";s:4:"name";s:10:"paul-
coles";s:3:"acl";s:1:"2";s:5:"email";s:16:"paul@passage.htb";s:4:"nick";s:10:"Paul

Coles";s:4:"pass";s:64:"e26f3e86d1f8108120723ebe690e5d3d61628f4130076ec6c
b43f16f497273cd";s:3:"lts";s:10:"154www-data@passage:/var/www/html/
CuteNews/cdata/users$

a:1:{s:4:"name";a:1:{s:9:"kim-swift";a:9:{s:2:"id";s:10:"1592483309";s:4:"name";s:9:"kim-
swift";s:3:"acl";s:1:"3";s:5:"email";s:15:"kim@example.com";s:4:"nick";s:9:"Kim

Swift";s:4:"pass";s:64:"f669a6f691f98ab0562356c0cd5d5e7dcdc20a07941c86adcfce9af30
85fbeca";s:3:"lts";s:10:"1592487094www-data@passage:/var/www/html/CuteNews/
cdata/users$

a:1:{s:4:"name";a:1:{s:5:"admin";a:8:
{s:2:"id";s:10:"1592483047";s:4:"name";s:5:"admin";s:3:"acl";s:1:"1";s:5:"email";s:17:"nad
av@passage.htb";s:4:"pass";s:64:"7144a8b531c27a60b51d81ae16be3a81cef722e11b43a

26fde0ca97f9e1485e1";s:3:"lts";s:10:"1592487988";s:3:"ban";s:1:"0";s:3:"cnt";s:)www-
data@passage:/var/www/html/CuteNews/cdata/users$

a:1:{s:4:"name";a:1:{s:6:"egre55";a:11:
{s:2:"id";s:10:"1598829833";s:4:"name";s:6:"egre55";s:3:"acl";s:1:"4";s:5:"email";s:15:"egre
55@test.com";s:4:"nick";s:6:"egre55";s:4:"pass";s:64:"4db1f0bfd63be058d4ab04f18f6533
1ac11bb494b5792c480faf7fb0c40fa9cc";s:4:"more";s:60:"YToyOntzOjQ6InNpbase64:
invalid input
```

Me guardo el hash de paul en un fichero y lo crackeo con hashcat

`hashcat -m 1400 paul.hash tools/wordlist/rockyou.txt`

![](/assets/images/Passage-Hackthebox/hashcat.png)

e26f3e86d1f8108120723ebe690e5d3d61628f4130076ec6cb43f16f497273cd:atlanta1

Y con `su paul` me convierto en el usuario paul.

![](/assets/images/Passage-Hackthebox/paul.png)

Ya puedo leer el user.txt

## Escalada al usuario navdav

A continuación me pongo a enumerar lo que hay en la home de paul y veo que dentro de .ssh hay claves privadas. Las miro y veo que son del usuario nadav!

Simplemente copio el contenido de la clave privada y lo pego en un fichero local.
Luego cambio los permisos para que no me de error con `chmod 600 id_rsa` y me conecto con `ssh -i id_rsa nadav@10.10.10.206`

![](/assets/images/Passage-Hackthebox/navdav.png)

## Escalada a root

Llegado a este punto empiezo enumerando los ficheros de la home del usuario y veo en .ICEauthority que se repite varias veces el texto MAGIC-COOKIE.
Busco información sobre eso (iceauthority privilege escalation) y encuentro una página en la que explican un bug que permite crear ficheros con permisos de root.

[https://unit42.paloaltonetworks.com/usbcreator-d-bus-privilege-escalation-in-ubuntu-desktop/](https://unit42.paloaltonetworks.com/usbcreator-d-bus-privilege-escalation-in-ubuntu-desktop/)

Ya que puedo crear ficheros con privilegios de root, me creo en mi máquina un par de claves ssh con ssh-keygen. Después copio el contenido de mi clave pública y me creo un fichero en la máquina passage con el contenido de mi clave.

Por último ejecuto

```
gdbus call --system --dest com.ubuntu.USBCreator --object-path /com/ubuntu/USBCreator --method com.ubuntu.USBCreator.Image /dev/shm/key /root/.ssh/authorized_keys true
```

para copiar mi clave pública en root/.ssh/authorized_keys :)

Sólo me queda conectarme por ssh con el usuario root, ya que no me va a pedir contraseña.

![](/assets/images/Passage-Hackthebox/root.png)

Ya puedo leer el root.txt
