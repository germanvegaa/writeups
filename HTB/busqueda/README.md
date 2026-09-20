# Busqueda - HackTheBox

**Fecha de resolución:** 19 de septiembre de 2026

Máquina Linux, IP 10.129.59.241, hostname searcher.htb.

## Reconocimiento

```
nmap --privileged -n -vvv -sV -oN scan.txt 10.129.59.241
```

```
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.52
Service Info: Host: searcher.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Enumeración web

El sitio es un buscador que reenvía la consulta a distintos motores externos (Google, Accuweather, etc.) según el motor elegido. Capturé la petición del formulario para analizarla con calma:

```
POST /search HTTP/1.1
Host: searcher.htb
Content-Type: application/x-www-form-urlencoded

engine=Accuweather&query=a
```

## Explotación

Por debajo, la aplicación usa la librería Python Searchor en su versión 2.4.0 para construir la URL de cada motor a partir de la query, y lo hace pasando la entrada del usuario directamente a un `eval()`. Eso permite salir del contexto de cadena e inyectar código Python arbitrario en el parámetro `query`.

Con un payload que cierra la cadena esperada por Searchor y ejecuta un comando del sistema (por ejemplo, forzando una reverse shell con `os.system(...)` o `subprocess`), obtuve ejecución de código en el servidor. La shell recibida corresponde al usuario `svc`.

## Post-explotación

En el directorio de la aplicación desplegada había un `.git` con la configuración remota apuntando a una instancia local de Gitea (`127.0.0.1:3000`), y con credenciales para el usuario `administrator` de esa instancia guardadas en la config. Con acceso al Gitea interno revisé los repositorios privados, donde apareció un script de mantenimiento (`system-checkup.py`) junto a las credenciales reales del usuario `svc` del sistema, lo que me permitió confirmar el acceso persistente por SSH como `svc`.

## Escalada a root

`sudo -l` como `svc` muestra permiso para ejecutar ese mismo script como root:

```
(ALL) NOPASSWD: /usr/bin/python3 /opt/scripts/system-checkup.py
```

El script, al ejecutarse con el argumento `full-checkup`, invoca `full-checkup.sh` sin ruta absoluta, resolviéndolo contra el `PATH` del proceso. Creando mi propio `full-checkup.sh` malicioso y anteponiendo su directorio al `PATH` antes de lanzar el script por sudo, mi script se ejecuta con privilegios de root en lugar del original. Shell de root obtenida y flag capturada.

---

GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
