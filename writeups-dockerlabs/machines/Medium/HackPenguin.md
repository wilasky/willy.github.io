

> Habilidades: XXXxxXXXs (linux), esteganografía.
> 
![intro]()

# RECONOCIMIENTO

## Nmap Scan

Realizamos el primer sondeo a la IP de la víctima para averiguar qué puertos están abiertos:

~~~ bash
nmap --open -p- -n -sS -Pn $ip -oG FirsScan
~~~

![Primer_Escaneo]()

- `--open`: Mostrar los puertos abiertos.
- `-p-`: Escanear todo el rango de puertos (65535)
- `-sS`: Escaneo TCP SYN, técnica más sigilosa para determinar el estado del puerto.
- `-n`: No aplicar resolución DNS.
- `-Pn`: Deshabilitar el descubrimiento de host, asumiendo que el objetivo se encuentra activo.
- `-oG`: Exportar el escaneo a un formato `Grepable`, util para extraer información.

_____________________________________________________________________________________________________________________________________________________________________

### Enumerar Servicios
