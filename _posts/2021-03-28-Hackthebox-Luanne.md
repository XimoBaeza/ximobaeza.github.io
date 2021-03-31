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

![](/assets/images/Luanne-Hackthebox/puerto80.png)

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
![](/assets/images/Luanne-Hackthebox/shell.png)





