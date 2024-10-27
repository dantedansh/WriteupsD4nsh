
## Escaneo de puertos y servicios con nmap

`sudo nmap -sS --min-rate 5000 --open -vvv -n -Pn -p- 10.10.11.32 -oG Puertos`

![image](/assets/images/Maquinas/Sightless/ports.png)

Vemos estos puertos abiertos 21(ftp), 22(ssh) y 80(http).

Ahora lanzaremos scripts de enumeración bajo estos puertos:

`sudo nmap -sC -sV -p 21,22,80 10.10.11.32 -oN targeted`

![image](/assets/images/Maquinas/Sightless/scan.png)

Vemos que tiene un FTP llamado proFTP, y en la parte del puerto 80 nos da un indicio de que se usa un subdominio `sightless.htb` por lo que lo agregaremos al **/etc/hosts**:

![image](/assets/images/Maquinas/Sightless/addDNS.png)

<br>
## Enumerando el servidor web

Después de ver la web, decidí ver su código fuente y me encontré con otro subdominio:

![image](/assets/images/Maquinas/Sightless/sql.png)

Y lo agregamos al **/etc/hosts**:

![image](/assets/images/Maquinas/Sightless/addDNS2.png)

<br>
## Explotando la vulnerabilidad de sqlpad

Una vez agregado nos dirigimos a la ruta:

![image](/assets/images/Maquinas/Sightless/about.png)

Al parecer nos carga una plataforma que administra bases de datos en su versión 6.10.0, y buscando en la web dimos con que es vulnerable al siguiente CVE: [exploit CVE-2022-0944](https://github.com/shhrew/CVE-2022-0944)
Así que una vez clonamos el repositorio, seguimos las instrucciones del README, después vamos a ejecutarlo:

![image](/assets/images/Maquinas/Sightless/exploit.png)

Una vez ejecutado el script en base a sus instrucciones, hemos recibido la reverse shell:

![image](/assets/images/Maquinas/Sightless/docker.png)

Entramos a una maquina, pero esta debe ser una especie de contenedor o algo así ya que aquí no está ninguna flag, por lo que aún tenemos que seguir investigando.

<br>
## Escalando a otro usuario

Después de un rato cómo root recordé el archivo **/etc/shadow** dónde se almacenan los hash de ciertos usuarios, y revisando me topé con estos:

![image](/assets/images/Maquinas/Sightless/hashes.png)

Y vemos el hash de un usuario llamado michael, por lo que usando la herramienta john intentaremos crackear ese hash:

Primero identificaremos que tipo de hash es:

![image](/assets/images/Maquinas/Sightless/sha512.png)

Vemos que es un **SHA-512**, así que usando john, intentaremos crackear este hash:

`john --format=sha512crypt hash.txt --wordlist=/usr/share/wordlists/rockyou.txt`

![image](/assets/images/Maquinas/Sightless/password.png)

Vemos que logramos crackear el hash, y usando ssh nos conectamos cómo el usuario michael:

![image](/assets/images/Maquinas/Sightless/ssh.png)

Y vemos que nos hemos conectado.

<br>
## Escalando privilegios

Ahora que estamos dentro, estuve buscando con las cosas básicas:

![image](/assets/images/Maquinas/Sightless/find.png)

pero no encontré nada interesante a primera vista, así que me dedique a revisar los puertos abiertos internamente:

`netstat -tuln`

![image](/assets/images/Maquinas/Sightless/8080.png)

Hay algunos puertos, pero me llama la atención el 8080, ya que generalmente por defecto ese puerto es un servicio HTTP, así que lo que haremos será redirigir ese puerto hacía nuestra maquina atacante.

### Redirigiendo el puerto 8080 a nuestra maquina atacante

Para ello haremos lo siguiente en una nueva terminal:

`ssh michael@sightless.htb -L 127.0.0.1:8080:127.0.0.1:8080`

Lo que hará esto es que usando el parámetro "-L" de ssh nos permitirá agregar la dirección que queremos redirigir, en este caso la dirección local de la maquina en su puerto 8080 hacía nuestra maquina atacante en el puerto 8080, una vez hecho esto iremos a la web y vemos lo siguiente:

![image](/assets/images/Maquinas/Sightless/froxlor.png)

Al parecer es un servicio de control de servidores, y viendo no encontré mucho así que buscando, encontré lo siguiente: [Explotar remote debugger](https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/chrome-remote-debugger-pentesting/)

Es una forma de ver valores de un servicio mal configurado en el modo debugger remoto, y recordé que la maquina tenía el chrome instalado entonces revise lo siguiente:

![image](/assets/images/Maquinas/Sightless/debugger.png)

Y vemos que se usan procesos que utilizan el remote debugger, por lo que siguiendo la guía de arriba, vamos a ir a esta sección de nuestro navegador:

`chrome://inspect/#devices`

![image](/assets/images/Maquinas/Sightless/remote.png)

Y nos muestra este panel, ahora lo que haremos será en otra terminal redirigir algún puerto de los que encontramos abiertos internamente para ver si alguno de ellos nos arroja el debugger que se está utilizando y si es que esta mal configurado podremos ver algo, primero redirigimos algún puerto de los que encontramos en la enumeración:

![image](/assets/images/Maquinas/Sightless/8080.png)

Por ejemplo el 3306:

`ssh michael@sightless.htb -L 127.0.0.1:3306:127.0.0.1:3306`

En el panel de la web damos en configure y agregamos la dirección y el puerto:

![image](/assets/images/Maquinas/Sightless/add.png)

Agregamos la dirección y el puerto, y habilitamos la opción de port forwarding, damos en done pero no nos muestra nada:

![image](/assets/images/Maquinas/Sightless/3306.png)

Así que ahora haremos lo mismo pero con otro puerto, y después de intentar el puerto 37697 me dio el siguiente resultado:

![image](/assets/images/Maquinas/Sightless/37697.png)

Así que ahora damos en inspect:

![image](/assets/images/Maquinas/Sightless/capture.png)

Y podemos ver la actividad, y vemos que un usuario ingreso sus credenciales, e inmediatamente cuando está dentro del panel admin ya logueado vamos a darle a el botón de detener:

![image](/assets/images/Maquinas/Sightless/stop.png)

Ahora tenemos las consultas guardadas ya que hemos pausado esto y no perderemos los datos, así que revisando las consultas, encontramos las credenciales gracias al debugger:

![image](/assets/images/Maquinas/Sightless/credentials.png)

Y con esto podemos ya acceder al panel admin de la web principal:

![image](/assets/images/Maquinas/Sightless/admin.png)

Ahora que ya tenemos acceso, vemos que hay una sección de PHP:

![image](/assets/images/Maquinas/Sightless/php.png)

Así que desde aquí al parecer podemos configurar nuestra versión de PHP y que se ejecute:

![image](/assets/images/Maquinas/Sightless/command.png)

Estamos creando una versión de PHP que nos copie la flag root y que nos la pase a un .txt en la ruta /tmp, así que guardamos, y ahora vamos a System > settings:

![image](/assets/images/Maquinas/Sightless/fpm.png)

Y vemos que aquí se encuentra la configuración del PHP-FPM dónde hicimos la versión nueva con nuestro comando malicioso, así que entramos:

Y vamos a apagar el servicio:

![image](/assets/images/Maquinas/Sightless/off.png)

Y lo volvemos a encender:

![image](/assets/images/Maquinas/Sightless/on.png)

Con esto el archivo PHP se va a ejecutar ya que reiniciamos el PHP-FPM, y esperamos unos segundos y revisamos la ruta /tmp:

![image](/assets/images/Maquinas/Sightless/denied.png)

Al parecer nos da un error de lectura ya que no tenemos permisos, pero le daremos usando otra ejecución de versión PHP:

![image](/assets/images/Maquinas/Sightless/perm.png)

Hacemos el mismo proceso de después de agregar la instrucción reiniciar el PHP-FPM, y ahora después de unos segundos ya podremos leer la flag root:

![image](/assets/images/Maquinas/Sightless/root.png)

Y con esto hemos terminado la maquina c: