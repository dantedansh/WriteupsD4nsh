
## Escaneo con Nmap y whatweb

Empezaremos a escanear si existen puertos abiertos usando **nmap**:

```sudo nmap -sS --min-rate 5000 --open -vvv -n -Pn -p- -oG Puertos 10.129.40.18```

-sS: Enviaremos paquetes TCP SYN scan para comprobar si esta abierto o no.

--min-rate 5000: Enviar al menos 5 mil paquetes por segundo, esto es para acelerar el proceso.

--open: Que nos muestre solo los puertos abiertos.

-vvv: Queremos triple verbose para ver más información a medida que se va ejecutando y ver en tiempo real las respuestas.

-n: Desactivamos resolución DNS para acelerar el proceso de escaneo.

-Pn: Desactivar escaneo de hosts.

-oG: Guardar la salida del comando en un archivo llamado Puertos en formato Grep.

Y el resultado nos dice que los puertos:

![image](/assets/images/Maquinas/GreenHorn/ports.png)

Vemos que tenemos 3 puertos abiertos.

Ahora correremos scripts de enumeración bajo estos puertos:

`sudo nmap -sC -sV -p 22,80,3000 10.129.40.18 -oN targeted`

-sC : ejecutar scripts de enumeración.

-sV : Enumerar versión y servicio que corren bajo los puertos dados.

-oN : Guardando el resultado en formato nmap.

![image](/assets/images/Maquinas/GreenHorn/scripts.png)

Vemos información y nos damos cuenta que el puerto 3000 corre el servicio HTTP.

## Investigando los servidores web

Primero investigaremos el puerto 80:

![image](/assets/images/Maquinas/GreenHorn/redirect.png)

Al parecer debemos agregar en host "greenhorn.htb" para poder acceder a el, así que lo hacemos agregándolo en el archivo "/etc/hosts":

![image](/assets/images/Maquinas/GreenHorn/DNS.png)

Ahora que hemos agregado el host, vamos a ver que hay en la web:

![image](/assets/images/Maquinas/GreenHorn/web.png)

Vemos que nos carga la siguiente web, al investigar a simple vista nos damos cuenta que debajo nos dice que está usando un CMS llamado "pluck".

## Fuzzing web

usaremos la herramienta **cwfuzz** la cuál es una herramienta para hacer fuzzing que ha creado mi amigo **c4rta** y el repositorio de la herramienta está aquí: [CWFUZZ]("https://github.com/ic4rta/cwfuzz")


`cwfuzz -u http://greenhorn.htb/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x 200,301 -t 200 -e php,txt`

-u : Indicamos la URL a fuzzear.

-w : Indicamos la ruta del diccionario.

-x : Indicamos que código de estado queremos que nos muestre solamente.

-t : Cantidad de threads(hilos).

-e : Extensiones que deseamos ver de archivos, en este caso php y txt.

![image](/assets/images/Maquinas/GreenHorn/cwfuzz.png)

Podemos ver que nos ha encontrado algunos archivos, vemos que hay un panel login, entre algunos otros.

Revisando el login.php encontramos lo siguiente:

![image](/assets/images/Maquinas/GreenHorn/pluck.png)

Nos lleva a ingresar una contraseña, la cuál no conocemos y buscando contraseñas por defecto para pluck en su versión 4.7.18 probamos "password", "admin", entre otras pero ninguna dio resultado.

Así que tocará seguir buscando.

<br>

Ahora revisaremos el siguiente servidor web, el que corre bajo el puerto 3000:

![image](/assets/images/Maquinas/GreenHorn/git.png)

Encontramos lo siguiente, vemos que es un servicio que usa Git, y explorando encontramos lo siguiente:

![image](/assets/images/Maquinas/GreenHorn/repo.png)

Encontramos el siguiente repositorio, y viendo el código del archivo login.php:

![image](/assets/images/Maquinas/GreenHorn/ruta.png)

Podemos ver que utiliza un archivo llamado pass.php, el cuál usa para comparar, por lo que al revisar en la ruta de ese archivo, encontramos lo siguiente:

![image](/assets/images/Maquinas/GreenHorn/hash.png)

Nos topamos con el siguiente hash, así que ahora intentaremos crackearlo.

## Crackeando el hash encontrado

![image](/assets/images/Maquinas/GreenHorn/sha-512.png)

Vemos que con la herramienta **hash-identifier** nos dice que lo más posible es que el hash este usando el algoritmo "SHA-512".

Por lo que vamos a intentar crackearlo:

`john --format=raw-sha512 hash.txt --wordlist=/usr/share/wordlists/rockyou.txt`

![image](/assets/images/Maquinas/GreenHorn/crack.png)

Hemos conseguido crackear el hash.

## Explotando una vulnerabilidad RCE

Así que ahora probaremos iniciar sesión en el login.php:

![image](/assets/images/Maquinas/GreenHorn/access.png)

Podemos ver que hemos logrado entrar, y buscando en **searchsploit**:

![image](/assets/images/Maquinas/GreenHorn/rce.png)

Encontramos que existe un exploit para ejecución remota de comandos RCE.

Y al examinar el código de ese exploit vemos lo siguiente:

![image](/assets/images/Maquinas/GreenHorn/code.png)

Al parecer lo que se intenta en el exploit es acceder a una sección de la web donde se puede instalar un módulo, que al parecer se sube en formato ZIP. y al final vemos que se apunta hacía la ruta del modulo miri.php.

Así que al parecer la web puede cargar un zip luego lo extrae y podemos apuntar al php interpretando las instrucciones que queramos.

<br>
Lo primero que haremos será obtener una reverse shell en php, yo obtuve la siguiente de aquí: [Pentest Monkey PHP reverse shell](https://pentestmonkey.net/tools/web-shells/php-reverse-shell)

Ahora descomprimimos el archivo .tar, y editaremos el archivo .php que viene dentro, para indicar que queremos que se conecte a nuestro equipo atacante por medio de un puerto que dejaremos en escucha:

![image](/assets/images/Maquinas/GreenHorn/change.png)

Podemos ver que hemos agregado nuestra IP atacante de VPN y también el puerto 4444 ya que ese es el que dejaré en escucha para ver si recibo alguna conexión.

Guardamos la reverse shell php y la vamos a meter a un archivo .zip:

![image](/assets/images/Maquinas/GreenHorn/zip.png)

Podemos ver que ya tenemos la reverse shell en un archivo .zip, ahora iremos a la ruta de la web donde nos indica que podemos subir nuestros módulos en formato ZIP:

![image](/assets/images/Maquinas/GreenHorn/install.png)

Y aquí vamos a cargar nuestro archivo ZIP:

![image](/assets/images/Maquinas/GreenHorn/upload.png)

Ya se ha subido, ahora vamos a ponernos en escucha antes de apuntar al archivo:

![image](/assets/images/Maquinas/GreenHorn/listener.png)

Ahora apuntamos al archivo:

![image](/assets/images/Maquinas/GreenHorn/url.png)

Y en el listener recibimos la reverse shell:

![image](/assets/images/Maquinas/GreenHorn/reverse.png)

Podemos ver que hemos accedido al servidor, y en la ruta /home vemos los siguientes usuarios:

![image](/assets/images/Maquinas/GreenHorn/users.png)

Intentando migrar al usuario "junior" con la contraseña que crackeamos anteriormente logramos acceder a su usuario:

![image](/assets/images/Maquinas/GreenHorn/junior.png)

Ahora estamos como el usuario junior, pero hay que hacer el tratamiento de la TTY para tener una SHELL interactiva.

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

<br>
## Escalando privilegios a root

![image](/assets/images/Maquinas/GreenHorn/flag.png)

Podemos ver que hemos logrado la primera flag, y enseguida vemos un PDF, lo vamos a descargar, crearemos un servidor HTTP con python para rápidamente poder acceder a ese archivo:

`python3 -m http.server 8888 &`

![image](/assets/images/Maquinas/GreenHorn/server.png)

![image](/assets/images/Maquinas/GreenHorn/pdf.png)

Ahora podemos acceder y descargar el PDF desde la web, aunque también podríamos hacerlo con wget.

Una vez tenemos el archivo:

![image](/assets/images/Maquinas/GreenHorn/mv.png)

Le he cambiado el nombre al PDF para que sea más simple tratarlo.

El PDF nos muestra lo siguiente:

![image](/assets/images/Maquinas/GreenHorn/censored.png)

Nos indica que podemos usar openvas como super usuario pero probamos y no existe el binario, por lo que iremos por otro camino, vemos que nos muestra un ejemplo de ingresar contraseña pero esta censurado, esta vez ya viene así censurado, yo no lo he hecho.

Así que intentaremos descifrar si usaron alguna herramienta insegura para censurar eso, primero vamos a extraer la imagen del PDF con la herramienta **pdfimages** que se instala con `sudo apt install poppler-utils`

`pdfimages -png vas.pdf output.png`

![image](/assets/images/Maquinas/GreenHorn/extract.png)

Y ya tenemos la imagen separada:

![image](/assets/images/Maquinas/GreenHorn/see.png)

Ahora utilizaremos una herramienta llamada **Depix** que descifra el texto si es que se uso un programa para censurar débil, la herramienta es la siguiente: [Depix](https://github.com/spipm/Depix)

Ahora la usaremos cómo nos indica en el README:

![image](/assets/images/Maquinas/GreenHorn/use.png)

En el parámetro -p va la ruta de la imagen que queremos descifrar.
El -s lo dejamos así.
Y en el -o indicamos la ruta donde queremos que se guarde el resultado en caso de poder descifrar lo pixeleado.

![image](/assets/images/Maquinas/GreenHorn/loading.png)

Ahora nos esta cargando, debemos esperar un rato, y al ir a el resultado obtuvimos una imagen clara y pudimos ver la contraseña:

![image](/assets/images/Maquinas/GreenHorn/password.png)

Podemos ver la contraseña, obviamente ahora la tuve que censurar, y al poner esta contraseña al usuario root hemos logrado escalar privilegios:

![image](/assets/images/Maquinas/GreenHorn/root.png)

Y con esto hemos terminado con esta maquina c: