
> Habilidades: Enumeración, steganografía, escalada de privilegios (linux)
_____________________________________________________________________________________________________________________________________________________________________

# RECONOCIMIENTO
_____________________________________________________________________________________________________________________________________________________________________
## Nmap Scan
_____________________________________________________________________________________________________________________________________________________________________
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

_____________________________________________________________________________________________________________________________________________________________________

### Enumerar Servicios
_____________________________________________________________________________________________________________________________________________________________________
Realizamos enumaeración a los puertos abiertos.
~~~ bash
nmap -sVC -p 22,80 172.17.0.2 -oN ServicesScan
~~~

![Escaneo_Sevicios](https://i.imgur.com/6ybGn1A.png)
- `-sC`: Ejecuta scripts NSE (Nmap Scripting Engine) predeterminados.
- `-sV`: Detecta la versión del servicio que se está ejecutando en cada puerto abierto.
_____________________________________________________________________________________________________________________________________________________________________

## Http Service
_____________________________________________________________________________________________________________________________________________________________________

Vemos el puerto `80` que corresponde a un servicio `http`, veamos que hay en él. Podemos usar la herramienta `whatweb` para listar las tecnologías detectadas en el servidor. (Como alternativa, puedes usar la extension de navegador Wappalyzer)

~~~ bash
whatweb http://172.17.0.2
~~~

![whatweb](https://i.imgur.com/iz3f6TK.png)

Abrimos en navegador y vamos a dicha web, leemos y podemos ver algo interesante, una nota de `Carlota`.
Yo siempre reviso view-source de la web por si se me escapa algo interesante.
![despido](https://i.imgur.com/JTVpkU4.png)

_____________________________________________________________________________________________________________________________________________________________________

## Fuzzing
_____________________________________________________________________________________________________________________________________________________________________

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
_____________________________________________________________________________________________________________________________________________________________________

## Fuerza Bruta
_____________________________________________________________________________________________________________________________________________________________________
Como obtuvimos dos nombres anteriormente, vamos a intenta hacer fuerza bruta con uno de ellos. En este caso usaremos dos herramientas diferentes aunque con el mismo resultado.

~~~ bash
hydra -l carlota -P /usr/share/wordlists/rockyou.txt -t4 ssh://172.17.0.2
~~~

![hydra](https://i.imgur.com/gHUFhTq.png)


- `-l`: Especifica un nombre.
- `-P`: Especifica una lista de palabras.
- `-t4`: Especifica hilos a usar. (más rapido)
- `ssh`: Especifica el protocolo.

~~~ bash
medusa -h 172.17.0.2 -u carlota -P /usr/share/wordlists/rockyou.txt -M ssh -t 4
~~~
![gobuster](https://i.imgur.com/phRZEFc.png)

- `-h`: Especifica la dirección IP del objetivo.
- `-u`: Especifica el nombre de usuario a probar.
- `-P`: Especifica el archivo de lista de contraseñas a utilizar.
- `-M`: Especifica el módulo a utilizar, en este caso SSH.
- `-t 4`: Especifica el número de hilos a utilizar para agilizar el proceso.

__¡¡BINGO!! obtuvimos las credenciales de Carlota.__

_____________________________________________________________________________________________________________________________________________________________________
# Escalada de privilegios
_____________________________________________________________________________________________________________________________________________________________________

Una vez tenemos acceso, tratamos la TTY para poder usarla comodamente, para esto cambiaremos el valor de la variable `$TERM`. En este caso, nuestra shell (SSH) ya es interactiva, solo necesitaremos export TERM=xterm.

~~~ bash
> control Z
stty raw -echo; fg
reset xterm
export TERM=xterm
stty size
stty rows 44 columns 184


~~~

## Enumeracion entorno

Primeramente veremos si tenemos privilegios `sudo` con el siguiente comando:

~~~ bash
sudo -l
~~~

![sudo -l]()

A continuación, intentar enumerar todo lo que los privilegios del usuario pueda. Si a primera vista no enconramos nada, habría que hacerlo mediante comandos.
Este caso observamos un archivo sospechoso, antes de seguir buscando vamos a analizar el archivo.
