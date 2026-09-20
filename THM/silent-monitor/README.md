# Silent Monitor - TryHackMe

**Fecha de resolución:** 26 de agosto de 2026

Máquina Linux, IP 10.130.161.178. Panel interno llamado "CorpNet — Network Operations Centre".

## Reconocimiento

```
nmap --privileged -sS -sV -sC -n -vvv -p22,5050 -oN scan.txt 10.130.161.178
```

SSH y un servicio web sobre Werkzeug/Flask en el puerto 5050.

## Enumeración web

```
gobuster dir -u http://10.130.161.178:5050 -w /usr/share/wordlists/dirb/common.txt
```

Aparece una ruta `internal` accesible directamente, parte del panel de NOC. Desde ahí se llega a una base de datos KeePass (`infrastructure.kdbx`) con credenciales de la infraestructura.

## Cracking

Extraje el hash de la base con `keepass2john` (`kdb.hash`, `kdb2.hash`; el primer intento de extracción quedó vacío y tuve que repetirlo). Crackeado el hash, abrí la base de datos y obtuve credenciales válidas para uno de los usuarios del sistema (`jmartin`, `netops` o `svc-mon`, según `users.txt`), con las que accedí por SSH.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
