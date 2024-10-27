
## Escaneo de puertos abiertos con nmap

Empezaremos escaneando los puertos que están abiertos dentro de la maquina:

`sudo nmap -sS --min-rate 5000 --open -vvv -n -Pn -p- 10.10.11.20 -oG Puertos`

![image](/assets/images/Maquinas/Editorial/puertos.png)

Vemos que existen 2 puertos abiertos, el 22 (SSH) y el 80 (HTTP).

Ahora vamos a escanear la versión y servicios que corren bajo estos puertos:

`sudo nmap -sC -sV -p 22,80 10.10.11.20 -oN targeted`

![image](/assets/images/Maquinas/Editorial/scan.png)

Vemos que no nos arroja la gran cosa, así que ahora vamos a investigar la web.

## Revisando el sitio web

Primero usamos la herramienta **whatweb** para ver con que posible cosas nos encontraremos:

`whatweb http://10.10.11.20/`

![image](/assets/images/Maquinas/Editorial/domain.png)

Podemos ver que tratamos con un server Ubuntu, usa nginx 1.18.0, etc. Y también vemos que está haciendo **virtual hosting** hacía la dirección editorial.htb, así que agregaremos en DNS a el archivo **/etc/hosts**:

![image](/assets/images/Maquinas/Editorial/addDNS.png)

Lo hemos agregado, y ahora iremos a la web y nos topamos con la siguiente web:

![image](/assets/images/Maquinas/Editorial/books.png)

## Investigando en subida de archivos

Vemos que es una web sobre libros o algo relacionado, vemos una pestaña llamada "Publish with us", y nos muestra la ruta de "/upload":

![image](/assets/images/Maquinas/Editorial/upload.png)

Intentaremos subir un archivo cualquiera, en este caso hice un archivo .txt con el mensaje de "prueba", y veremos que pasa al subirlo, y al hacerlo se subió correctamente sin ningún cambio notorio aparente a simple vista.

Así que ahora intentaremos meter algún archivo que no sea .txt, por ejemplo un .php:

![image](/assets/images/Maquinas/Editorial/php.png)

Creamos este archivo y al subirlo vemos lo siguiente:

![image](/assets/images/Maquinas/Editorial/errimg.png)

Podemos ver que al parecer muestra una especie de error en el formato o algo similar, y al hacer clic derecho y en "abrir imagen en nueva pestaña" nos manda a una pestaña donde se alojan los archivos pero al volver a intentar ya no aparecía esa imagen de error.

## Detectando SSRF

Así que probando un método de SSRF(server side request forgery) detectamos que es vulnerable.

Primero abrí un servidor php en mi maquina atacante por el puerto 80:

`php -S 0.0.0.0:80`

![image](/assets/images/Maquinas/Editorial/phpserver.png)

Y ahora en el campo de poner nombre de la sección upload, pondremos la IP de nuestra maquina atacante por el puerto 80:

![image](/assets/images/Maquinas/Editorial/ssrf.png)

Al momento de enviar la petición vemos en el log de nuestro server que se realizó una petición por el método GET:

![image](/assets/images/Maquinas/Editorial/peticion.png)

Podemos ver que posiblemente sea vulnerable ya que hay una conexión en un campo dónde no debería haberla.

Entonces regresando a la web vemos cómo cambió la imagen de portada a error de imagen:

![image](/assets/images/Maquinas/Editorial/err2.png)

Así que damos clic derecho y abrir imagen en nueva pestaña:

![image](/assets/images/Maquinas/Editorial/notfound.png)

Y nos dice que no existe pero en la URL podemos ver que nos lleva a una ruta /static/uploads/. Aquí se deben alojar los elementos subidos.

Cómo sabemos que hay una conexión al poner una IP en el lugar del nombre, pondremos la localhost para que la petición se haga una petición así misma, pero está vez al capturar la petición la enviaremos al **repeater**:

![image](/assets/images/Maquinas/Editorial/request.png)

Vemos que nos responde con una URL, al viajar a ella nos topamos con lo siguiente, que es la URL de la imagen por defecto que nos muestra en la portada de la subida de archivos.

## Creando un payload para detectar puertos abiertos internamente

Ahora que sabemos que podemos hacer peticiones hacía la misma maquina usaremos esto, ya que generalmente al escanear puertos por fuera muchas veces no se detectan por firewalls que rechazan paquetes que no provienen dentro de su mismo servidor, pero cómo vamos a hacer peticiones desde el mismo servidor web el firewall no filtrará esto ya que no le resultará sospechoso.

Lo que queremos hacer es descubrir si hay algún puerto abierto, por lo que usaremos cómo base la conexión que tenemos por medio del valor del nombre del  libro, y en base a la respuesta veremos si hay algún valor fuera de lo común entre todos los puertos y esto podría indicar algo.

Primero vamos a enviar la petición al **intruder**:

![image](/assets/images/Maquinas/Editorial/intruder.png)

Primeramente damos a el botón de clear debajo de add, esto para eliminar los payloads por defecto, y ahora vamos a agregar y seleccionar el primer puerto, ya que esté será nuestra base para empezar a hacer peticiones de los puertos totales:

![image](/assets/images/Maquinas/Editorial/add.png)

Al seleccionar el primer puerto y dar en Add veremos que se nos puso el payload en ese valor:

![image](/assets/images/Maquinas/Editorial/payload.png)

> Nuestro payload se encierra en medio de 2 simbolos que parecen cómo una serpiente el primero indica el inicio del payload y el segundo el final, en este caso solo nos interesa un objetivo por eso el tipo sniper.

Ahora vamos a la pestaña de "payloads":

![image](/assets/images/Maquinas/Editorial/configpayload.png)

Una vez lo hayamos configurado, iniciaremos el ataque con "start attack", y esperaremos.

Después de un rato de peticiones filtramos por length longitud, y vemos algo extraño:

![image](/assets/images/Maquinas/Editorial/length.png)

podemos ver que está petición tiene 217 de longitud, cuando las demás tienen 227, por lo que significa que algo hay ahí que hizo responder de una forma diferente y esta respuesta se dio en el puerto 5000.

Así que ahora en la web de uploads vamos a ver que respuesta nos da al poner ese puerto:

`http://127.0.0.1:5000`

![image](/assets/images/Maquinas/Editorial/puerto5000.png)

Damos a abrir imagen en nueva pestaña:

Y nos descarga el siguiente archivo automáticamente:

![image](/assets/images/Maquinas/Editorial/data.png)

Usamos la herramienta `jq` para ordenar en sintaxis estos valores:

![image](/assets/images/Maquinas/Editorial/jq.png)

Estás rutas al parecer contienen archivos con datos, y donde encontré información útil fue en el "new_authors", así que me dirigí a esa ruta:

Ponemos esto en lugar del nombre:

`http://127.0.0.1:5000/api/latest/metadata/messages/authors`

Y nuevamente nos pasa lo de la imagen con error, damos clic derecho y en abrir en nueva pestaña y nos descarga automáticamente otro archivo:

![image](/assets/images/Maquinas/Editorial/credentials.png)

Vemos que nos da unas credenciales para acceder al sistema.

Y por medio de SSH entramos:

![image](/assets/images/Maquinas/Editorial/dev.png)

<br>
## Escalando privilegios

Ahora vamos a ver cómo escalar privilegios a root, en la ruta del usuario dev encontré la siguiente ruta:

![image](/assets/images/Maquinas/Editorial/git.png)

Vemos que hay un archivo .git, en la guía de Linux explique que hay maneras de ver registros en caso de que existan, entramos a la ruta .git y ejecutamos el comando `git log`:

![image](/assets/images/Maquinas/Editorial/logs.png)

Esto nos muestra todos los logs que ha habido en este repositorio, con el comando `git show <commit>` podemos ver los valores de cada cambio, y en un log encontré lo siguiente:

![image](/assets/images/Maquinas/Editorial/prod.png)

Vemos que el usuario prod fue reemplazado por las credenciales del usuario dev en algún momento, así que usamos las credenciales de prod para migrar a dicho usuario:

![image](/assets/images/Maquinas/Editorial/suprod.png)

Vemos que ahora hemos escalado a el usuario prod, ahora desde aquí veremos que más encontramos para llegar a root.
<br>
## Escalando privilegios a root

Empezando la investigación rápidamente encontré lo siguiente:

![image](/assets/images/Maquinas/Editorial/script.png)

Un script al cuál tenemos permisos de ejecutar como root.

![image](/assets/images/Maquinas/Editorial/code.png)

Este script lo que al parecer hace es clonar el repositorio que le des, y también notamos que importa Repo desde la librería git.

Buscando encontré que para usar esa librería se debe tener instalado "GitPython" y buscando en google me di cuenta que hay versiones vulnerables de GitPython, para ver la versión que tenemos de GitPython en la maquina usamos el comando:

`pip3 list`:

![image](/assets/images/Maquinas/Editorial/gitpython.png)

Vemos que tenemos la versión 3.1.29, y buscando exploits para está versión creamos la siguiente linea:

`sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c cat% /root/root.txt% >% /tmp/root'`

Primero decimos que ejecute el python3 cómo sudo, luego damos la ruta del script al que tenemos permiso de ejecutar cómo root, recuerda que debe ser con sus rutas absolutas, por último pasamos cómo parámetro el exploit que lo que va a hacer es hacernos un cat a el root.txt y nos lo va a poner dentro de la ruta /tmp.

Una vez hecho esto ya tendremos la flag de root:

![image](/assets/images/Maquinas/Editorial/root.png)

Y de esa forma tenemos ejecución remota de comandos para hacer lo que sea necesario, nuestro objetivo era la flag y ya la tenemos así que hemos terminado con esta maquina c: