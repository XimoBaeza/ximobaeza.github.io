---
layout: single
title: Ready - Hack The Box
excerpt: "Resolución de la máquina *Ready* de Hack The Box, una máquina Linux de dificultad media según la plataforma en la cual nos aprovechamos de una vulnerabilidad en una versión antigua de Gitlab para obtener el acceso inicial, para posteriormente elevar privilegios a root dentro de un contenedor docker y finalmente escapar del contenedor y obtener la shell de root en la máquina."
date: 2021-05-15
classes: wide
header:
  teaser: /assets/images/Ready-Hackthebox/ready.png
  teaser_home_page: true
categories:
  - HackTheBox
tags:
  - Web Exploiting
  - Privilege Escalation
  - Pentesting
  - Linux
---

# Máquina Ready de HackTheBox

![](/assets/images/Ready-Hackthebox/ready.png)

Resolución de la máquina *Ready* de Hack The Box, una máquina Linux de dificultad media según la plataforma en la cual nos aprovechamos de una vulnerabilidad en una versión antigua de Gitlab para obtener el acceso inicial,     para posteriormente elevar privilegios a root dentro de un contenedor docker y finalmente escapar del contenedor y obtener la shell de root en la máquina.

## Escaneo con nmap
---

```bash
> nmap -sC -sV -p22,5080 10.10.10.220 -oN targeted                                                                                                                                                              ─╯
Starting Nmap 7.80 ( https://nmap.org ) at 2021-04-23 16:48 CEST
Nmap scan report for 10.10.10.220
Host is up (0.32s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
5080/tcp open  http    nginx
| http-robots.txt: 53 disallowed entries (15 shown)
| / /autocomplete/users /search /api /admin /profile
| /dashboard /projects/new /groups/new /groups/*/edit /users /help
|_/s/ /snippets/new /snippets/*/edit
| http-title: Sign in \xC2\xB7 GitLab
|_Requested resource was http://10.10.10.220:5080/users/sign_in
|_http-trane-info: Problem with XML parsing of /evox/about
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 38.08 seconds
```

Vemos que hay un Gitlab en le puerto 5080.

![gitlab.png](:/c742c9e612b54ed7b94cc4d4b5b28f16)
