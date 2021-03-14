---
layout: single
title: Zerologon en Windows Server 2019
excerpt: "En éste artículo atacaremos un Windows Server 2019 con el rol de Active Directory utilizando Zerologon para obtener los hashes de las contraseñas de todos los usuarios y máquinas del
dominio. Quiero recalcar que para realizar éste ataque no necesitamos
credenciales de ningún tipo ni estar unidos al dominio, lo que hace de ésta
vulnerabilidad una de las más críticas encontradas hasta la fecha. Teniendo el
hash de la contraseña del administrador del dominio utilizaremos la conocida
técnica de pass the hash para obtener acceso al sistema con la cuenta del
administrador."
date: 2021-03-13
classes: wide
header:
  teaser: /assets/images/Zerologon/zerologon.jpg
  teaser_home_page: true
categories:
  - Windows
tags:
  - Windows
  - Privilege Escalation
  - Pentesting
---

![](/assets/images/Zerologon/zerologon.jpg)

En éste artículo atacaremos un Windows Server 2019 con el rol de Active Directory utilizando Zerologon para obtener los hashes de las contraseñas de todos los usuarios y máquinas del dominio. Quiero recalcar que para realizar éste ataque no necesitamos credenciales de ningún tipo ni estar unidos al dominio, lo que hace de ésta vulnerabilidad una de las más críticas encontradas hasta la fecha. Teniendo el hash de la contraseña del administrador del dominio utilizaremos la conocida técnica de pass the hash para obtener acceso al sistema con la cuenta del administrador.

Utilizaremos una máquina virtual con Linux para la parte del atacante y otra
con Windows Server 2019 y Active Directory para la víctima.

![](/assets/images/Zerologon/windows-server.png)

En hackplayers explican en profundidad la vulnerabilidad.

[https://www.hackplayers.com/2020/09/zerologon-desatado-comprometer-DCs-facilmente.html](https://www.hackplayers.com/2020/09/zerologon-desatado-comprometer-DCs-facilmente.html)

Citando el artículo de hackplayers:

Zerologon se aprovecha de una vulnerabilidad en el servicio Netlogon, que se utiliza para diversas tareas relacionadas con la autenticación de usuarios y máquinas en controladores de dominio.

Netlogon utiliza un protocolo criptográfico basado en AES-CFB8 que define IVs fijos de 16 bytes de ceros en lugar de IVs aleatorios. Con esto, dada una clave aleatoria, hay una probabilidad de 1 entre 256 de que el cifrado AES de un bloque de todo ceros de como salida todo ceros. Dado que las cuentas de equipo no se bloquean después de intentos de inicio de sesión no válidos, simplemente podemos intentarlo varias veces hasta que obtengamos esa clave y la autenticación sea exitosa. La media esperada de intentos necesarios será de 256, lo que en la práctica sólo lleva unos tres segundos. Con este método, podemos iniciar sesión como cualquier ordenador del dominio. Esto incluye controladores de dominio de respaldo e incluso el propio controlador de dominio de destino. Conseguiremos cambiar la contraseña del equipo controlador de dominio a una cadena vacía (todo ceros) para obtener privilegios y poder obtener los hashese de todos los usuarios de Active Directory. Después de vulnerar el equipo podemos volver a cambiar la contraseña de la cuenta de equipo controlador de dominio para que todo vuelva a funcionar correctamente. Ahora el DC vuelve a comportarse normalmente y el atacante se ha convertido en administrador de dominio.

## Testeando si el servidor es vulnerable

La empresa Secura publicó una herramienta para identificar si una máquina es
vulnerable a Zerologon en un repositorio de github [https://github.com/SecuraBV/CVE-2020-1472](https://github.com/SecuraBV/CVE-2020-1472)

Nos clonamos la herramienta con `git clone
https://github.com/SecuraBV/CVE-2020-1472.git` y la lanzamos contra nuestro
controlador de dominio virtualizado, que se llama DC01 y tiene la ip
192.168.160.131

![](/assets/images/Zerologon/testing.png)

Y vemos que nos dice que es vulnerable a zerologon. Ésta vulnerabilidad afecta
a todas las versiones de Windows Server desde la 2008 R2, siempre que no
cuenten con las últimas actualizaciones instaladas. Microsoft sacó un parche en agosto, pero por lo visto había que editar el registro a mano para asegurarse de que el sistema quedaba completamente parcheado. En concreto la clave del registro es FullSecureChannelProtection y tiene que tener un valor de “1”. Se supone que en febrero tenía que salir la actualización que parcheaba definitivamente la vulnerabilidad, con lo cual deben de haber muchos servidores a dia de hoy que siguen siendo vulnerables.

El nombre del servidor y el del dominio se pueden averiguar rápidamente con
crackmapexec.

![](/assets/images/Zerologon/info.png)

## Explotando la vulnerabilidad y accediendo al sistema

Existen varias pruebas de concepto que explotan zerologon pero yo en este caso
voy a utilizar ésta: [https://github.com/dirkjanm/CVE-2020-1472](https://github.com/dirkjanm/CVE-2020-1472)

Igual que antes, nos clonamos el repositorio con `git clone
https://github.com/dirkjanm/CVE-2020-1472.git`

![](/assets/images/Zerologon/clone.png)

Y ahora ejecutamos el script en python, pero antes vamos a crear un entorno
virtual con virtualenv para que no nos den problemas los scripts de impacket.

Para ello ejecutamos 
```
virtualenv zerologon
source zerologon/bin/activate
```

Ahora nos vamos al directorio opt y nos clonamos la suite de herramientas de
impacket

```
git clone https://github.com/SecureAuthCorp/impacket
cd impacket
sudo pip install .
```

Ahora si, ejecutamos el exploit y vemos que funciona y nos dice Exploit
complete!

![](/assets/images/Zerologon/exploit.png)

Ahora ejecutamos el script de impacket secretsdump para obtener los hashes de
las contraseñas. Le pasamos un hash que corresponde a todo ceros para la cuenta
de máquina del controlador del dominio, por eso ponemos dom01.local/DC01$ con
un signo de dolar al final.

![](/assets/images/Zerologon/secrets.png)

Obtenemos tanto los hashes de las cuentas de usuario como los de las cuentas de
máquinas. Ésto es importante para poder después de ganar acceso al sistema
restablecer la contraseña de la cuenta de equipo y dejarla como estaba para que
todo siga funcionando correctamente.

Para usar pass the hash podemos utilizar wmiexec y obtendremos una sesión de
cmd de la cuenta del administrador del dominio.

![](/assets/images/Zerologon/pth.png)

Vemos que al ejecutar whoami nos devuelve dom01\administrador. Estamos dentro
de la máquina con la cuenta del administrador del dominio. En éste punto
podemos hacer lo que queramos, por ejemplo crearnos un usuario administrador
oculto para que no se vea desde windows, activar el escritorio remoto editando
el registro con los comandos reg y conectarnos por RDP al servidor.

Pero no nos olvidemos de que tenemos que restablecer la contraseña de la cuenta
de quipo para que todo funcione correctamente a nivel de active directory. Esto
es debido a que hemos cambiado la contraseña de la cuenta del equipo a nivel de
active directory, pero en la sam local sigue estando la original.

Para restablecerla usaremos de nuevo secretsdump pero esta vez con el hash del
administrador, lo que nos devolverá la *plain_password_hex* original para poder
restablecerla con el script restorepassword.py

```
secretsdump.py dom01.local/administrador@192.168.160.131 -target-ip 192.168.160.131 -hashes :57987b38d5ba54e02daa2d5d7579765b
```

Nos devuelve entre otras cosas la plain_password_hex de la cuenta de equipo en
la sam local.

```
DOM01\DC01$:plain_password_hex:fc3547348ef0ff05e8427168cd24be642dc407bf1d352de423fe0651b653ce853d7e37c957bd504444cd6898b301f56c00061671e25c11ddcf3890ef8b8bfcc41c8672fa708c4aa5981d8a1006288e7938887597eef55c630b37d5ddcd5635237f0b9be59944353f7ac163f60ad134c4f9506f768c49a33beda57a039e2a50d2d022c50a3d488998ee7f6cf2861c82bba15a67121c896fd479dc2688a6a2400b3a0c49ae1af1909672033929927e38e91fbe1a01e6d4ee15a381b6ccc42cdcc578fad4630161d429bc63fc7376e5debc403afba544390b42592408bb44c47e6da38b111d98d9cb9fadbfa89bb2936f53
```

Solo quedaría ejecutar el script para restablecer la contraseña local de la
cuenta de equipo.

```
sudo python3 restorepassword.py dom01.local/DC01@DC01 -target-ip 192.168.160.131 -hexpass fc3547348ef0ff05e8427168cd24be642dc407bf1d352de423fe0651b653ce853d7e37c957bd504444cd6898b301f56c00061671e25c11ddcf3890ef8b8bfcc41c8672fa708c4aa5981d8a1006288e7938887597eef55c630b37d5ddcd5635237f0b9be59944353f7ac163f60ad134c4f9506f768c49a33beda57a039e2a50d2d022c50a3d488998ee7f6cf2861c82bba15a67121c896fd479dc2688a6a2400b3a0c49ae1af1909672033929927e38e91fbe1a01e6d4ee15a381b6ccc42cdcc578fad4630161d429bc63fc7376e5debc403afba544390b42592408bb44c47e6da38b111d98d9cb9fadbfa89bb2936f53
```

![](/assets/images/Zerologon/restore.png)

Para comprobar si todo ha ido bien podemos ejecutar de nuevo nuestra primera
ejecución de secretsdump y esta vez debería dar error al no estar la contraseña
de la cuenta del equipo vacía.


