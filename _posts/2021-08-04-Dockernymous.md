---
layout: single
title: Dockernymous - Anonimizando contenedores docker
excerpt: "En este artículo veremos como utilizando el script dockernymous.sh podemos hacer que todo el tráfico hacia internet que genere un contenedor sea redirigido a la red tor. El script levanta dos contenedores, uno lo utilizaremos como *gateway* y incluye las reglas de iptables especificadas en la página oficial del proyecto de tor [https://gitlab.torproject.org/legacy/trac/-/wikis/doc/TransparentProxy](https://gitlab.torproject.org/legacy/trac/-/wikis/doc/TransparentProxy) para anonimizar el tráfico del otro contenedor, que lo utilizaremos como *workstation*, al más puro estilo *whonix*."
date: 2021-08-04
classes: wide
header:
  teaser: /assets/images/Dockernymous/tor2.jpg
  teaser_home_page: true
categories:
  - Docker
tags:
  - Anonymous
  - Linux
---

![](/assets/images/Dockernymous/tor2.jpg)

En este artículo veremos como utilizando el script dockernymous.sh podemos hacer que todo el tráfico hacia internet que genere un contenedor sea redirigido a la red tor. El script levanta dos contenedores, uno lo utilizaremos como *gateway* y incluye las reglas de iptables especificadas en la página oficial del proyecto de tor [https://gitlab.torproject.org/legacy/trac/-/wikis/doc/TransparentProxy](https://gitlab.torproject.org/legacy/trac/-/wikis/doc/TransparentProxy) para anonimizar el tráfico del otro contenedor, que lo utilizaremos como *workstation*, al más puro estilo *whonix*.

Empezamos clonando el repositorio que se encuentra en la siguiente URL: [Dockernymous](https://github.com/bcapptain/dockernymous).
```bash
git clone https://github.com/bcapptain/dockernymous
```

Creamos una red específica para los contenedores a la que llamaremos docker_internal.

```bash
docker network create --driver=bridge --subnet=192.168.0.0/24 docker_internal
```

Ahora descargamos una imagen de alpine, para que sea lo más ligera posible, que será la que utilizaremos para *gateway*.

```bash
docker pull alpine
```

Ejecutamos el contenedor e instalamos los paquetes necesarios.

```bash
docker run -it alpine /bin/sh
apk add --update tor iptables iproute2
```

En este punto yo quise instalar el paquete *nyx*, que es monitor del tráfico de tor en tiempo real, y que también sirve para muchas más cosas. Recomiendo echar un vistazo a la página del proyecto [Nyx](https://nyx.torproject.org/).

Para instalar nyx opté por utilizar pip, así que instalé primero pip en el contenedor y luego nyx.

```bash
apk add --update py3-pip
pip install nyx
exit
```

Seguidamente ejecutaremos ```docker ps -a``` y nos quedaremos con el ID del contenedor, para luego ejecutar ```docker commit [Container ID] my_gateway``` y crear nuestra imagen cuyo nombre será my_gateway con los cambios que hemos hecho aplicados. Si queremos cambiar el nombre de la imagen podemos hacerlo, pero tendríamos que cambiarlo también en el script dockernymous.sh para que funcione.

Ahora vamos con la imagen que hará las funciones de *workstation*. En mi caso utilizaré kali linux, que tiene una imagen docker bastante ligera para que podamos luego instalar sólamente los paquetes que necesitemos.

```bash
docker pull kalilinux/kali-rolling
```
Ahora instalaremos los paquetes que queramos. Con ```apt search kali-tools``` podemos ver las distintas categorías que tiene kali destinadas a pentesting, y con ```apt-cache show kali-tools-categoria | grep Depends``` podemos ver los paquetes que pertenecen a esa categoría para instalar solo los que necesitemos. Por ejemplo ```apt-cache show kali-tools-exploitation | grep Depends```.

```bash
docker run -it kalilinux/kali-rolling /bin/bash
apt update
apt full-upgrade
apt install iproute2 net-tools ...
apt clean
exit
```

Me encontré con que el contenedor no tenía el comando ip y fallaba el script dockernymous.sh al intentar obtener la ip. De ahí el instalar el paquete iproute2 y net-tools. Lo comento porque el autor de la herramienta lo omite en su README.

Cuando ya tengamos instalado todo lo que necesitemos creamos la nueva imagen.

```bash
docker commit [Container ID] my_workstation

```

Tuve que modificar una línea del script, en concreto la 67, en la que hace un curl para obtener nuestra ip pública, ya que el sitio web no devuelve nada actualmente y al no poder obtener la ip daba un error.

Cambié ```curl --silent https://canihazip.com/s``` por ```curl --silent https://ifconfig.me/ip``` y solucioné el error.

Solo nos falta ejecutar el script como root y ya se encargará de levantar los dos contenedores y enrutar el tráfico hacia tor.

![](/assets/images/Dockernymous/dockernymous-test2.png)

El autor de la herramienta sugiere instalar xfce4 y vnc para conectarnos al entorno gráfico del contenedor, esto ya lo dejo a vuestra elección.
