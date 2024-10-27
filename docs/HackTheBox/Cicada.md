
## Escaneo de puertos y servicios con nmap

Empezamos escaneando los puertos abiertos con nmap:

`sudo nmap -sS --min-rate 5000 --open -vvv -n -Pn -p- 10.10.11.35 -oG Puertos`

![image](/assets/images/Maquinas/Cicada/puertos.png)

Vemos que tenemos una gran cantidad de puertos abiertos, así que lanzaremos un escaneo de enumeración con nmap para ver que versión y servicios corren bajo esos puertos:

![image](/assets/images/Maquinas/Cicada/servicios.png)

Viendo los puertos podemos ver que está el servicio de smb, ldap, kerberos, entre otros.

## Investigando el servicio SMB

Primero haremos un listado como usuario no autenticado para ver que rutas existen en el SMB:

`smbclient -L //10.10.11.35/ -U anonymous`

![image](/assets/images/Maquinas/Cicada/smb.png)

Vemos multiples rutas de recursos compartidos, entre ellos están: ADMIN, C, DEV ,HR ,IPC, NETLOGON, SYSVOL.

Intentando acceder al único que me dejo ingresar fue al directorio HR, una vez dentro listamos con dir:

![image](/assets/images/Maquinas/Cicada/notice.png)

Y encontramos un documento de texto, vamos a descargarlo con el comando get:

`get "Notice from HR.txt"`

Y ahora en nuestra maquina tendremos descargado el archivo:

![image](/assets/images/Maquinas/Cicada/hr_pass.png)

Encontramos un instructivo y unas credenciales, ahora hay que averiguar que podemos lograr con estas credenciales.

## Usando la herramienta nxc para enumerar usuarios

Esta herramienta **nxc** nos sirve para enumerar multiples protocolos, tanto como smb, ldap y winrm.

Vamos a tratar de enumerar usuarios usando esta herramienta:

`nxc smb 10.10.11.35 -u Guest -p '' --rid-brute > salida.txt`

smb : Indicamos que queremos en el modo de smb.

10.10.11.35 : La IP del host victima.

-u Guest : Muchos smb tienen usuarios por defecto cómo invitado y Guest esta entre los comunes.

-p ' ' : Por lo general los usuarios invitados no ocupan password, por lo que se deja vacía.

--rid-brute : Indicamos que se haga un ataque de fuerza bruta de la herramienta para enumerar recursos y también redirigimos la salida a un archivo llamado salida.txt.

Una vez hecho esto, en caso de que el servicio SMB disponga de un usuario Guest, la herramienta nos dejará en el archivo de salida lo siguiente:

![image](/assets/images/Maquinas/Cicada/output.png)

Nos interesa solo quedarnos con los valores que son usuario, o sea los que tienen la etiqueta (SidTypeUser), podemos usar esta cadena para filtrar por expresiones regulares los que nos interesan:

Hice esta expresión regular para filtrar por los usuarios que nos interesan y guardar la salida en users.txt:

`cat salida.txt | grep "SidTypeUser" | awk 'NF{print $(NF-1)}' | cut -d '\' -f 2 > users.txt`

Ahora que tenemos esta lista de usuarios haremos lo siguiente.

## Ataque de fuerza bruta al servicio SMB para un "login"

Recordamos que tenemos una password que encontramos, ahora usaremos ataque de fuerza bruta usando esa contraseña contra esos usuarios para ver si a alguno le pertenece esa password, usaremos nuevamente la herramienta **nxc**:

`nxc smb 10.10.11.35 -u users.txt -p 'Ci............' --continue-on-success`

smb : que se conecte al servicio smb del host "10.10.11.35".

-u users.txt : Diccionario de usuarios que usará para probar con:

-p 'Ci...........' : Password que encontramos y queremos validar con los usuarios filtrados.

![image](/assets/images/Maquinas/Cicada/michael.png)

Podemos ver que el usuario **michael.wrightson** dio positivo antes este ataque.

Ahora que ya conocemos el usuario y password de dicho usuario bajo el servicio smb, vamos a averiguar que accesos tiene.

## Verificando permisos que tiene el usuario encontrado

Para ello nuevamente usaremos la herramienta **nxc**:

Probamos listar si tenía permisos en el smb:

`nxc smb 10.10.11.35 -u 'michael.wrightson' -p 'Ci...........' --shares`

--shares : Para ver los recursos compartidos y sus permisos.

Pero esto no nos arrojo nada. 

## Enumerando el servicio ldap

así que en vez de enumerar el servicio smb, vamos a probar con el ldap:

`nxc ldap 10.10.11.35 -u michael.wrightson -p 'Ci.........' -M get-desc-users`

ldap : Conectarnos al servicio ldap del host **10.10.11.35**.

-u michael.wrightson : Nos autenticaremos cómo este usuario.

-p 'Ci..........' : Con su respectiva contraseña.

-M get-desc-users : Nos da una lista de los usuarios disponibles con una descripción.

Y nos arroja esto:

![image](/assets/images/Maquinas/Cicada/david_pass.png)

Nos da un usuario llamado **david.orelious** y en su descripción dejo una contraseña, así que ya tenemos otras credenciales.

Ahora con este usuario veremos si tiene acceso a recursos del smb:

`nxc smb 10.10.11.35 -u 'david.orelious' -p 'aR........' --shares`

![image](/assets/images/Maquinas/Cicada/shared_smb_david.png)

Vemos que tiene acceso de lectura en el directorio DEV del servidor, al cuál no teniamos acceso anteriormente, así que nos conectaremos al servidor SMB con sus credenciales:

![image](/assets/images/Maquinas/Cicada/backup_script.png)

Podemos ver que existe un archivo una especie de script, lo descargamos con get, y al verlo nos encontramos con lo siguiente:

![image](/assets/images/Maquinas/Cicada/emily_pass.png)

Las credenciales de un usuario llamado **emily.oscars**.

## Accediendo al servidor (primera flag)

Recordemos que el servicio de winrm estaba habilitado, y cómo estas credenciales no me llevaron a nada más, probando ponerlas en winrm con la herramienta **evil-winrm**:

`evil-winrm -i 10.10.11.35 -u emily.oscars -p 'Q!...........'`

![image](/assets/images/Maquinas/Cicada/access.png)

Y la primera flag la encontramos:

![image](/assets/images/Maquinas/Cicada/flag.png)

<br>
## Escalando privilegios..... (pendiente).