# Operation Coldstart - TryHackMe

**Fecha de resolución:** 27 de agosto de 2026

Máquina Linux, IP 10.128.139.232. Sitio corporativo de "Volt Labs".

## Reconocimiento

```
nmap --privileged -sS -n -vv -sV -sC -p21,22,80 -oN scan.txt 10.128.139.232
```

FTP (vsftpd 3.0.5) con login anónimo permitido, SSH y un servicio web sobre Gunicorn ("URL Preview - Volt Labs").

## FTP anónimo

```
ftp 10.128.139.232
```

Con usuario `anonymous` se accede al directorio `pub`, donde había una copia de seguridad (`backup.tar.gz`). La descargué y extraje su contenido (`backup`).

## Credenciales filtradas

Dentro del backup apareció una nota interna con credenciales de acceso a staging:

```
=== INTERNAL ===
SSH access for staging:
  user: webdev
  pass: V0ltLabs#summer
- Mara
```

## Acceso

```
ssh webdev@10.128.139.232
```

con esas credenciales, quedando con acceso al sistema como `webdev`.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
