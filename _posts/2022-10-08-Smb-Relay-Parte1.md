---
layout: single
title: Smb Relay - Parte 1
excerpt: "Para hacer este artículo me he montado un pequeño laboratorio con dos máquinas windows 11 unidas a un controlador de dominio con windows server 2022, para recrear una auditoría interna en la que empezamos con la captura de credenciales usando la herramienta responder, para luego hacer un reenvío de esas credenciales a la segunda máquina, obtener acceso a ésta, ver las credenciales guardadas en el navegador, hacer un volcado de memória del proceso lsass.exe y transferirlo a nuestro equipo, para ver las credenciales del administrador del dominio con pypykatz y conectarnos al servidor con evil-winrm. Además generaremos una persistencia con golden ticket para conectarnos al controlador de dominio cuando queramos sin necesidad de utilizar contraseña."
date: 2022-10-08
classes: wide
header:
  teaser: /assets/images/Smb-Relay/responder2.png
  teaser_home_page: true
categories:
  - Ciberseguridad
tags:
  - Windows
  - Privilege Escalation
  - Pentesting
---

Para hacer este artículo me he montado un pequeño laboratorio con dos máquinas windows 11 unidas a un controlador de dominio con windows server 2022, para recrear una auditoría interna en la que empezamos con la captura de credenciales usando la herramienta responder, para luego hacer un reenvío de esas credenciales a la segunda máquina, obtener acceso a ésta, ver las credenciales guardadas en el navegador, hacer un volcado de memória del proceso lsass.exe y transferirlo a nuestro equipo, para ver las credenciales del administrador del dominio con pypykatz y conectarnos al servidor con evil-winrm. Además gereraremos una persistencia con golden ticket para conectarnos al controlador de dominio cuando queramos sin necesidad de utilizar contraseña.

## Descripción del laboratorio

### Controlador de dominio
 - SO: Windows Server 2022 Standard
 - Nombre: DC1
 - IP: 192.168.0.100
 - Dominio: SIETEREINOS.LOCAL

 ![](/assets/images/Smb-Relay/DC1.PNG)


### Equipo 1
 - SO: Windows 11 Profesional
 - Nombre: PC01
 - IP: 192.168.0.202
 - Usuario: SIETEREINOS\Jon (Jon Nieve)

![](/assets/images/Smb-Relay/PC01.png)

### Equipo 2
 - SO: Windows 11 Profesional
 - Nombre: PC02
 - IP: 192.168.0.169
 - Usuario: SIETEREINOS\Arya (Arya Stark)

![](/assets/images/Smb-Relay/PC02.png)

### Máquina atacante
 - Linux debian con todas las herramientas necesarias instaladas (responder, ntlmrelayx, proxychains, DonPapi, hoaxshell, pypykatz, evil-winrm, crackmapexec y la suite de impacket)

## Introducción

En entornos corporativos windows es muy común el uso de mecanismos de resolución de nombres basados en broadcast (LLMNR - Local Link Multicast Name Resolution, NBT-NS - NetBIOS Naming Services) y la activación por defecto de IPv6 en los servidores y estaciones de trabajo.

LLMNR y NBT-NS son mecanismos a los que el sistema operativo recurre de forma automática cuando a través de DNS no se obtiene ningún resultado. Es decir, que cada vez que se produce una petición de red por cualquier equipo, en la que el destino no existe se recurre a ellos, haciendo una petición broadcast a toda la red, y si la petición no va firmada, se confiará en cualquier respuesta que se reciba. Estas peticiones se producen muy a menudo por tareas de backup obsoletas, y otras tareas automatizadas que hacen peticiones a equipos que en ese momento estén apagados, por ejemplo, un servidor de actualizaciones que intenta actualizar un equipo que está apagado, un equipo que intenta conectar una unidad de red que ya no existe, o una impresora que ya no está compartida, por poner algunos ejemplos. Es en este punto cuando entra en juego la herramienta responder, que ante una petición de este tipo se va a hacer pasar por el recurso capturando las credenciales en formato hash netntlm, y si la contraseña es debil se podrá crackear fácilmente.

Además es posible hacer un reenvío de esas credenciales hacia otros equipos, y en el caso de que tengan privilegios sobre ese otro equipo podemos ganar acceso a ese segundo equipo. En este artículo veremos solo algunos ejemplos de como un atacante que ha logrado acceso a la red interna con, por ejemplo un phising o instalando un dispositivo físico para conectarse desde fuera, podría aprovecharse de ello.

## Captura de credenciales con Responder

Lo primero es enumerar los equipos que hay en la red y ver si tienen el smb firmado o si utilizan smb1 con crackmapexec (cme smb 192.168.0.0/24)

![](/assets/images/Smb-Relay/cme.png)

Vemos mucha información útil. El nombre de los equipos, el dominio, y también vemos que en el servidor el smb está firmado pero en los dos equipos restantes no. Y que ninguno de los 3 utiliza smb1.

Empezamos con la captura de credenciales con responder. Tan solo tenemos que poner en marcha el responder y esperar una petición a un recurso compartido que no exista.

El comando es responder -I enp3s0 -v (enp3s0 es mi interfaz de red, puede que la tuya sea otra)

![](/assets/images/Smb-Relay/responder.png)

Forzamos una petición a un recurso que no exista en la máquina PC01.

![](/assets/images/Smb-Relay/noexiste.png)

Y vemos como automáticamente el responder captura el hash de la autenticación.

![](/assets/images/Smb-Relay/hash.png)

Guardamos el hash en un fichero hash.txt y, si la contraseña es debil la podremos crackear fácilmente con hashcat.

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```
![](/assets/images/Smb-Relay/hashcat.png)

Ya tenemos un usuario y una contraseña. Usuario sietereinos\jon contraseña P@ssw0rd!1

## Reenvío de credenciales a otra máquina con ntlmrelayx

Vamos a suponer un escenario en el que el usuario jon de la máquina PC01 tiene privilegios de administrador en la máquina PC02. Lo que haremos será un reenvío de las credenciales capturadas con la petición del PC01, para que nos devuelva todos los hashes ntlm (no netntlm que son los que captura el responder) del PC02, y realizar otras acciones como ver las contraseñas que tiene el usuario Arya del PC02 guardadas en el navegador y finalmente ganar acceso a la máquina.

Los hashes netntlm que captura el responder no sirven para hacer pass-the-hash, pero se pueden crackear como hemos visto anteriormente. Y los ntlm si sirven para hacer pass-the-hash.

Para hacer el smb relay o reenvío de credenciales necesitamos hacer unos cambios en el archivo responder.conf de la siguiente forma:

```bash
sed -i 's/HTTP = On/HTTP = Off/g' /opt/tools/Responder/Responder.conf 
sed -i 's/SMB = On/SMB = Off/g' /opt/tools/Responder/Responder.conf 
```
Se trata de cambiar tanto el SMB como el HTTP a Off.

Ahora ejecutamos el responder como antes (responder -I enp3s0 -v) y ntlmrelayx (ntlmrelayx -tf target.txt -of netntlm -smb2support). Antes tenemos que crear un archivo target.txt con la ip del equipo que recibirá ese reenvío de credenciales, en mi caso la ip 192.168.0.169. Podemos sacar una lista de los equipos con smb sin firmar con el siguiente comando (cme smb 192.168.0.0/24 --gen-relay-list targets.txt).

Hacemos otra petición a un recurso que no exista desde el PC01 y vemos como se produce el reenvío y nos devuelve todos los hashes ntlm de la máquina PC02.

![](/assets/images/Smb-Relay/relay.png)

El comportamiento por defecto de ntlmrelayx es mostrar los hashes ntlm, pero podemos hacer que haga un túnel socks, para poder ejecutar comandos a través de ese túnel en la máquina PC02, aprovechándonos de ese reenvío de credenciales.

Hacemos lo mismo que antes pero le pasamos al ntlmrelayx el parámetro -socks (ntlmrelayx -tf target.txt -of netntlm -smb2support -socks)

Y vemos como se establece el túnel, con el comando socks desde la consola que hemos ejecutado el ntlmrelayx.

![](/assets/images/Smb-Relay/socks.png)

Podríamos generar una lista con crackmapexec (cme smb 192.168.0.0/24 --gen-relay-list targets.txt) y simplemente esperar a que se produzca un reenvío a algún equipo y cuando en alguno ponga admin status TRUE, como en nuestro ejemplo, es que las credenciales capturadas tienen privilegios de administrador en la máquina que recibe el reenvío, y podemos dumpear los hashes ntlm o hacer las siguientes acciones que veremos a continuación. 

Ahora configuramos el proxychains.conf para que utilice el proxy local por el puerto 1080 y todos los comandos que ejecutemos utilizando proxychains se autenticarán en la máquina PC02 con las credenciales capturadas del PC01.

Así podemos ejecutar DonPAPI.py que es un script que se encarga de dumpear credenciales almacenadas. En el PC02 el usuario Arya tiene almacenada en el firefox la contraseña de la página web donde se guardan las copias de seguridad de la empresa, por comodidad. Vamos a ver como sea cual sea, en este caso da igual lo compleja que sea porque DonPAPI la leerá en texto claro.

Ejecutamos proxychains DonPAPI.py -no-pass 'SIETEREINOS'/'JON'@'192.168.0.169' y obtenemos las credenciales.

![](/assets/images/Smb-Relay/donpapi.png)

En la parte 2 veremos como acceder tanto al PC02 como al controlador de dominio, y como obtener persistencia.

[SMB Relay Parte 2](https://ximobaeza.github.io/ciberseguridad/Smb-Relay-Parte2/)


