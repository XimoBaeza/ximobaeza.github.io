---
layout: single
title: Dockernymous - Anonimizando contenedores docker
excerpt: "En este artículo veremos como utilizando el script dockernymous.sh podemos hacer que todo el tráfico hacia internet que genere un contenedor sea redirigido a la red tor. El script levanta dos contenedores, uno lo utilizaremos como *gateway* y incluye las reglas de iptables especificadas en la página oficial del proyecto de tor para anonimizar el tráfico del otro contenedor, que lo utilizaremos como *workstation*, al más puro estilo *whonix*."
date: 2021-08-04
classes: wide
header:
  teaser: /assets/images/Dockernymous/tor.png
  teaser_home_page: true
categories:
  - Docker
tags:
  - Anonymous
  - Linux
---

![](/assets/images/Dockernymous/tor.png)

En este artículo veremos como utilizando el script dockernymous.sh podemos hacer que todo el tráfico hacia internet que genere un contenedor sea redirigido a la red tor. El script levanta dos contenedores, uno lo utilizaremo  s como *gateway* y incluye las reglas de iptables especificadas en la página oficial del proyecto de tor para anonimizar el tráfico del otro contenedor, que lo utilizaremos como *workstation*, al más puro estilo *whonix*.

Empezamos clonando el repositorio que se encuentra en la siguiente URL: [Dockernymous](https://github.com/bcapptain/dockernymous).
```bash
git clone https://github.com/bcapptain/dockernymous
```

Creamos una red específica para el script a la que llamaremos docker_internal.

```bash
docker network create --driver=bridge --subnet=192.168.0.0/24 docker_internal
```

Ahora descargamos una imagen de alpine, para que sea lo más ligera posible, que será la que utilizaremos para *gateway*.

```bash
docker pull alpine
```

Ejecutamos el contenedor y instalamos los paquetes necesarios.

```bash
docker run -it alpine /bin/sh
apk add --update tor iptables iproute2
```

En este punto yo quise instalar el paquete *nyx*, que es monitor del tráfico de tor en tiempo real, y que también sirve para más cosas. Recomiendo echar un vistazo a la página del proyecto [Nyx](https://nyx.torproject.org/).

Para instalar nyx opté por utilizar pip, así que instalé primero pip en el contenedor y luego nyx.

```bash
apk add --update py3-pip
pip install nyx
exit
```

Seguidamente ejecutaremos ```docker ps -a``` y nos quedaremos con el ID del contenedor, para luego ejecutar ```docker commit [Container ID] my_gateway``` y crear nuestra imagen cuyo nombre será my_gateway con los cambios que hemos hecho aplicados. Si queremos cambiar el nombre de la imagen podemos hacerlo, pero tendríamos que cambiarlo también en el script dockernymous.sh para que funcione.

