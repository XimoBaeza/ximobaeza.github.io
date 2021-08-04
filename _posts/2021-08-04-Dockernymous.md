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

