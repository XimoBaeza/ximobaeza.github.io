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

Zerologon se aprovecha de una vulnerabilidad en el servicio Netlogon, que se utiliza para diversas tareas relacionadas con la autenticación de usuarios y máquinas en controladores de dominio.

Netlogon utiliza un protocolo criptográfico basado en AES-CFB8 que define IVs fijos de 16 bytes de ceros en lugar de IVs aleatorios. Con esto, dada una clave aleatoria, hay una probabilidad de 1 entre 256 de que el cifrado AES de un bloque de todo ceros de como salida todo ceros. Dado que las cuentas de equipo no se bloquean después de intentos de inicio de sesión no válidos, simplemente podemos intentarlo varias veces hasta que obtengamos esa clave y la autenticación sea exitosa. La media esperada de intentos necesarios será de 256, lo que en la práctica sólo lleva unos tres segundos. Con este método, podemos iniciar sesión como cualquier ordenador del dominio. Esto incluye controladores de dominio de respaldo e incluso el propio controlador de dominio de destino. Conseguiremos cambiar la contraseña del equipo controlador de dominio a una cadena vacía (todo ceros) para obtener privilegios y poder obtener los hashese de todos los usuarios de Active Directory. Después de vulnerar el equipo podemos volver a cambiar la contraseña de la cuenta de equipo controlador de dominio para que todo vuelva a funcionar correctamente. Ahora el DC vuelve a comportarse normalmente y el atacante se ha convertido en administrador de dominio.


