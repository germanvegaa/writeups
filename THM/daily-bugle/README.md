# Daily Bugle - TryHackMe

**Fecha de resolución:** 31 de agosto de 2026

Máquina Linux, IP 10.129.187.173. El sitio es "The Daily Bugle", un Joomla montado sobre CentOS.

## Reconocimiento

```
nmap --privileged -sS -n -vvv -p22,80,3306 -sV -sC -oN scan.txt 10.129.187.173
```

Ports 22 (OpenSSH 7.4), 80 (Apache 2.4.6, PHP 5.6.40 sobre CentOS) y 3306 (MySQL). `configuration.php`, expuesta por Joomla, confirma que el sitio es "The Daily Bugle".

## Enumeración web

```
gobuster dir -u http://10.129.187.173/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x php,txt,bak
```

Directorios propios de Joomla (images, media, templates, modules) confirman la versión del CMS.

## Explotación

El parámetro `list[fullordering]` de `com_fields` es inyectable (SQLi). Con sqlmap:

```
sqlmap -u "http://10.129.187.173/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=" --dbms=mysql --level=5 --risk=3 --dump
```

sqlmap confirma el punto de inyección (error-based y time-based) y vuelca las tablas de Joomla, incluyendo el hash bcrypt del usuario admin (`hash.txt`). Lo crackeé con la wordlist de `word.txt` y obtuve la contraseña en texto claro (`spiderman123`), coherente con la temática del sitio.

Con esas credenciales entré al panel de administración de Joomla y, desde ahí, edité una plantilla PHP para inyectar una reverse shell, consiguiendo ejecución como el usuario del servicio web (`apache`/`jjameson` según el sistema).

## Escalada a root

`sudo -l` muestra que el usuario puede ejecutar `/usr/bin/yum` como root sin contraseña. Siguiendo la técnica de GTFOBins para yum, creé un plugin Python malicioso y un `.repo` temporal que yum carga al ejecutarse, logrando invocar `/bin/sh` con privilegios de root a través de la propia ejecución de yum.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
