# Internal - TryHackMe

**Fecha de resolución:** 2 de septiembre de 2026

Máquina Linux, IP 10.128.156.8.

## Reconocimiento

```
nmap --privileged -sS -sV -p22,80 -sC -oN scan.txt 10.128.156.8
```

SSH (OpenSSH 7.6p1) y un Apache 2.4.29 sirviendo la página por defecto de Ubuntu.

## Enumeración web

Detrás de la página por defecto hay un WordPress. Enumerando directorios apareció un backup de configuración (`wp-config.php`) con las credenciales de la base de datos (`wordpress`/`wordpress123`), aunque no reutilizables directamente para el login del CMS.

## Acceso inicial

Entre los archivos filtrados apareció una nota con credenciales de un usuario del sistema:

```
aubreanna:bubb13guM!@#123
```

que permiten acceso por SSH como `aubreanna`.

## Pivotando a Jenkins interno

Desde dentro, un escaneo de puertos internos muestra un Jenkins escuchando solo en la red de Docker (172.17.0.2:8080), no expuesto al exterior. Levanté un túnel SSH para llegar a él:

```
ssh -L 8080:172.17.0.2:8080 aubreanna@10.128.156.8
```

Contra el formulario de login de Jenkins (`j_acegi_security_check`) lancé hydra:

```
hydra -l admin -P /usr/share/wordlists/rockyou.txt -s 8080 -m /j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in:invalid 172.17.0.2 http-post-form
```

y obtuve `admin:spongebob`.

## Escalada a root

Con acceso de administrador a Jenkins, usé la Script Console para ejecutar Groovy y conseguir una shell dentro del contenedor Docker donde corre Jenkins. Desde ahí, revisando el sistema encontré las credenciales de root del host, que reutilicé para conectarme directamente como root a la máquina principal.

---

GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
