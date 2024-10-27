## Escaneo con Nmap

Empezaremos a escanear si existen puertos abiertos usando **nmap**:

```sudo nmap -sS --min-rate 5000 --open -vvv -n -Pn -p- -oG Puertos 10.10.11.23```

-sS: Enviaremos paquetes TCP SYN scan para comprobar si esta abierto o no.

--min-rate 5000: Enviar al menos 5 mil paquetes por segundo, esto es para acelerar el proceso.

--open: Que nos muestre solo los puertos abiertos.

-vvv: Queremos triple verbose para ver más información a medida que se va ejecutando y ver en tiempo real las respuestas.

-n: Desactivamos resolución DNS para acelerar el proceso de escaneo.

-Pn: Desactivar escaneo de hosts.

-oG: Guardar la salida del comando en un archivo llamado Puertos en formato Grep.

Y el resultado nos dice que los puertos:

![image](/assets/images/Maquinas/Permx/nmap.png)

Vemos el puerto 22(SSH) y el 80(HTTP) que están abiertos.

Revisaremos la web:

![image](/assets/images/Maquinas/Permx/DNS.png)

Vemos que nos rechaza la conexión, y nos dice el host **permx.htb** así que lo agregaremos al **/etc/hosts/**:

![image](/assets/images/Maquinas/Permx/hosts.png)

Y ahora entramos y nos topamos con la siguiente web:

![image](/assets/images/Maquinas/Permx/web.png)

Y buscando por la web visualmente no encontramos nada interesante.
<br>
## Fuzzing de la página web

Así que hicimos fuzzing usando la herramienta cwfuzz:

`cwfuzz -u http://permx.htb/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x 200,301 -t 200 -e php,txt`

![image](/assets/images/Maquinas/Permx/fuzzing.png)

Pero vemos que no nos está mostrando nada interesante, así que ahora haremos fuzzing de sub dominios:
<br>
## Fuzzing de sub dominios

Esta vez usaremos wfuzz:

`wfuzz -w subdomains-top1million-5000.txt --hc=302 -t 200 -H "Host: FUZZ.permx.htb" http://permx.htb/`

![image](/assets/images/Maquinas/Permx/lms.png)

Podemos ver que nos da un sub-dominio llamado **lms**, así que agregaremos ese sub-dominio al **/etc/hosts/**:

![image](/assets/images/Maquinas/Permx/subdns.png)

Y al acceder nos dirige a esta web:

![image](/assets/images/Maquinas/Permx/chamilo.png)
<br>
## Explotando la vulnerabilidad RCE

Y buscando en la web alguna vulnerabilidad para este servicio llamado chamilo, encontramos lo siguiente:

[Chamilo LMS <= 1.11.24 - Remote Code Execution](https://pentest-tools.com/vulnerabilities-exploits/chamilo-lms-11124-remote-code-execution_22949?source=post_page-----f2d1b348d7f8--------------------------------)

Nos dice que en la ruta `/main/inc/lib/javascript/bigupload/inc/bigUpload.php` existe una ruta en la cuál se pueden subir archivos php sin restricciones, y vemos que está habilitada esa ruta:

![image](/assets/images/Maquinas/Permx/ruta.png)

Así que buscando un exploit encontramos el siguiente:

[Exploit CVE-2023-4220](https://github.com/Ziad-Sakr/Chamilo-CVE-2023-4220-Exploit.git)

El exploit nos dice que su uso es el siguiente:

![image](/assets/images/Maquinas/Permx/usage_exploit.png)

Nos dice que primero debemos ejecutar el script, luego pasarle ciertos parámetros, ocupamos el -f para indicarle el archivo que  se subirá a la ruta insegura, en este caso será una shell reversa en PHP que conseguimos de aquí: [Pentest monkey php reverse shell](https://pentestmonkey.net/tools/web-shells/php-reverse-shell)

Ahora que descarguemos la reverse shell y la extraigamos vamos a modificar para que la conexión se hacía nuestra maquina atacante:

![image](/assets/images/Maquinas/Permx/change.png)

Vemos que asignamos nuestra IP de VPN de atacante, y la conexión será enviada a través del puerto 4444, guardamos.

Y ahora ya podemos poner esta reverse shell php en el parámetro que nos lo pedía, ahora el parámetro que sigue es el -h que es simplemente el HOST de donde se encuentra chamilo, y por último con el parámetro -p se pondrá el puerto que queramos poner en escucha, en este caso abriremos el 4444 ya que fue el que configuramos en el código de reverse shell de php.

Así que nuestra ejecución quedaría así:

`./CVE-2023-4220.sh -f php-reverse-shell.php -h http://lms.permx.htb/ -p 4444`

Y al ejecutar:

![image](/assets/images/Maquinas/Permx/reverse.png)

Podemos apreciar que hemos obtenido la conexión.

Así que ya estamos dentro del servidor, ahora tenemos que escalar privilegios.
<br>
## Escalando a un usuario con más privilegios

Ahora nos interesa ubicar los archivos de configuración de chamilo, ya que podríamos encontrar algún archivo con credenciales, buscamos si existe algún archivo de configuración de chamilo, y encontramos lo siguiente:

![image](/assets/images/Maquinas/Permx/conf.png)

Ahora sabemos que en algún lado del sistema existe un archivo llamado **configuration.php**, así que usando el comando find vamos a buscar desde la raíz del sistema este posible archivo filtrando por su nombre:

`find / -type f -name "configuration.php" 2>/dev/null`

![image](/assets/images/Maquinas/Permx/find.png)

Podemos ver que nos detecto 2 rutas, leeremos la primera:

Y entre todos los datos encontramos las siguientes credenciales de sesión:

![image](/assets/images/Maquinas/Permx/credentials.png)

Podríamos intentar entrar a la base de datos a ver si encontramos contraseñas y usuarios, pero primero probaremos usar esa contraseña con algún usuario del sistema disponible, vamos a la ruta /home/:

![image](/assets/images/Maquinas/Permx/mtz.png)

Vemos que existe el usuario **mtz** así que con la contraseña encontrada anteriormente intentaremos entrar a este usuario, ya que es muy común que la gente use la misma contraseña para muchas cosas diferentes, así que probaremos:

![image](/assets/images/Maquinas/Permx/escalar.png)

Vemos que estábamos en lo cierto, así que ahora ya podemos leer la primera flag:

![image](/assets/images/Maquinas/Permx/flag.png)
<br>
## Escalando privilegios a root

Una vez estamos cómo mtz, ahora debemos escalar a root, por lo que usando **sudo -l**, listaremos a que programas o archivos tenemos permisos de ejecución cómo root sin necesidad de usar contraseña, es decir, que podemos usar ese archivo con permisos root aunque no tengamos la contraseña, y viendo encontramos lo siguiente:

![image](/assets/images/Maquinas/Permx/sudo.png)

Así que veremos que contiene este script:

![image](/assets/images/Maquinas/Permx/code.png)

Así que lo primero que haremos será crear un enlace simbólico del archivo **/etc/sudoers** y este enlace se llamará **helpfile** y se guardará en la ruta actual.

`ln -s /etc/sudoers helpfile`

![image](/assets/images/Maquinas/Permx/helpfile.png)

Vemos que se ha creado este enlace, y esto lo hicimos ya que el archivo **/etc/sudoers** configura que usuarios pueden acceder cómo super usuario, entonces lo que sucede es que gracias a que el script que encontramos el cuál nos permite cambiar permisos de archivos que se encuentran dentro de /home/mtz, la ruta donde está nuestro enlace, y este enlace llamado **helpfile** apunta al archivo **/etc/sudoers**, y gracias al script que nos permite cambiar permisos, agregaremos que el archivo del enlace **helpfile** tenga permisos de lectura y escritura para el usuario mtz:

`sudo /opt/acl.sh mtz rw /home/mtz/helpfile`

Indicamos que se ejecute como usuario privilegiado usando el sudo, ponemos la ruta del script, luego usamos los parámetros los cuales primero es el usuario en este caso mtz, seguido de los permisos y el archivo al cuál se aplicarán esos permisos.

Entonces una vez hecho esto, podremos editar el archivo **/etc/sudoers/** usando vim:

![image](/assets/images/Maquinas/Permx/add.png)

Y vemos que hemos agregado al usuario mtz a la lista de usuarios que tienen privilegios de administrador, así que ahora simplemente guardamos, y ejecutamos:

`sudo su`

Y ahora podremos acceder como usuario privilegiado poniendo la contraseña del usuario mtz que previamente habíamos obtenido:

![image](/assets/images/Maquinas/Permx/root.png)

Y hemos conseguido escalar máximos privilegios, y con esto hemos terminado con esta maquina :D

