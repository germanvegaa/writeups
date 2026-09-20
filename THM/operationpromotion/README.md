# Operation Promotion - TryHackMe

**Fecha de resolución:** 27 de agosto de 2026

Máquina Linux, IP 10.130.182.121. Portal de empleo de "RecruitCorp".

## Reconocimiento

```
nmap --privileged -sS -n -sV -sC -p22,80,139,445 -oN scan.txt 10.130.182.121
```

Apache 2.4.58, Samba y un `robots.txt` que desaconseja indexar `/admin/`, señalando directamente el panel de administración.

## Enumeración

El portal expone una lista de empleados (`users.txt`: admin, mvasquez, tparker, lhayes, kchen, rdavis, sysmaint, jbailey, aokafor) y, filtrado en algún punto de la web, un fichero de configuración de base de datos sacado del control de versiones:

```
# RecruitCorp application database config
# Pulled out of source control - DO NOT COMMIT.
db_host=localhost
db_name=recruitcorp
db_user=jford
db_pass_hash=$2b$10$QzkXmGndA2cQLozO3xAN6eWKrl6ZXyzhYTJNF67exOmTmN5oVSEfq
db_engine=sqlite3
```

## Cracking

El hash bcrypt de `jford` no cae con rockyou, así que generé una wordlist dirigida a partir de patrones propios del usuario (`jfordpasslist.txt`) y terminé crackeándolo, obteniendo su contraseña en texto claro.

## Acceso

Con `jford` y la contraseña recuperada, acceso al panel `/admin/` del portal, que es donde de verdad vive la gestión de la aplicación (no está pensado para verse en la web pública, de ahí el `robots.txt`).

---

GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
