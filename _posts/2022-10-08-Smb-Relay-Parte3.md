---
layout: single
title: Smb Relay - Parte 3
excerpt: "En esta tercera parte veremos como configurar una gpo en el controlador de dominio para conseguir que las peticiones SMB vayan firmadas, deshabilitar los protocolos LLMNR y NBT-NS, y añadir una clave de registro para deshabilitar también mDNS y conseguir que el responder no pueda capturar tráfico de red en todo el dominio."
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

En esta tercera parte veremos como configurar una gpo en el controlador de dominio para conseguir que las peticiones SMB vayan firmadas, deshabilitar los protocolos LLMNR y NBT-NS, y añadir una clave de registro para deshabilitar también mDNS y conseguir que el responder no pueda capturar tráfico de red en todo el dominio.

## Mitigación de SMB Relay usando gpo

Lo primero que vamos a hacer es configurar una gpo, en mi caso crearé una nueva que se llamará SMB Relay.

![](/assets/images/Smb-Relay/admgpo.png)

Vamos a administración de directivas de grupo y, en objetos de directiva de grupo, damos botón derecho y creamos una nueva gpo con el nombre que queramos.

![](/assets/images/Smb-Relay/newgpo.png)

Luego botón derecho sobre la directiva creada y editar.

Primero vamos a Configuración de equipo, directivas, configuración de windows, configuración de seguridad, directivas locales, opciones de seguridad, y ahí habilitamos dos opciones para que las peticiones smb vayan firmadas.

 - Cliente de redes de Microsoft: firmar digitalmente las comunicaciones (siempre)
 - Servidor de red Microsoft: firmar digitalmente las comunicaciones (siempre)

![](/assets/images/Smb-Relay/smbsign1.png)
![](/assets/images/Smb-Relay/smbsign2.png)

A continuación habilitamos en Configuración de equipo, directivas, plantillas administrativas, red, cliente dns, desactivar resolución de nombres de multidifusión. Esta configuración deshabilita las peticiones LLMNR.

![](/assets/images/Smb-Relay/llmnr.png)

Seguídamente vamos a deshabilitar NetBIOS sobre TCP/IP. Esto lo haremos con un script en powershell ya que no existe una gpo específica para ello.

Creamos un script con el siguiente contenido:

```powershell

```

Con esto lo que hacemos es, guardar la ruta del registro en una variable llamada regkey, para luego recorrer todas las interfaces de red disponibles y agregar una clave llamada NetbiosOptions con valor 2.

![](/assets/images/Smb-Relay/script.png)

Lo guardamos por ejemplo como deshabilitar_netbios.ps1 y vamos a agregarlo a la gpo.

Vamos a Configuración del equipo, directivas, configuración de windows, scripts (inicio o apagado), inicio, y en scripts de powershell agregamos el script, con el parámetro -exec bypass.

![](/assets/images/Smb-Relay/scriptgpo.png)

![](/assets/images/Smb-Relay/editascript.png)

También ponemos Para este GPO, ejecute los scripts en el orde siguiente: Ejecutar los scripts de windows powershell al principio.

Y por último, tenemos que agregar una clave de registro para las peticiones mDNS. Para ello vamos a Configuración del equipo, preferencias, configuración de windows, registro, y ahí damos botón derecho, nuevo, elemento del registro. En subárbol ponemos HKEY_LOCAL_MACHINE, en ruta de la clave ponemos SYSTEM\CurrentControlSet\Services\Dnscache\Parameters. En nombre del valor EnableMDNS, Tipo de valor REG_DWORD, información del valor 0 y base hexadecimal.

![](/assets/images/Smb-Relay/reg.png)

La nueva GPO la podemos vincular al dominio entero, o solo a alguna unidad organizativa, dependiendo de como tengamos configurado el dominio, y aplicará las configuraciones en todos los equipos del dominio o de la unidad organizativa respectívamente.

Con estos cambios aplicados en los equipos la herramienta responder ya no podrá capturar las autenticaciones tal como hemos visto anteriormente.

En la parte 4 veremos como, a pesar de aplicar las mitigaciones, éstas se aplican a ipv4, y por ipv6 aún podemos realizar diferentes ataques.
