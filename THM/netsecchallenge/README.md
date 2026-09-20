# Network Security Challenge - TryHackMe

**Fecha de resolución:** 27 de agosto de 2026

Máquina Linux, IP 10.128.179.222, con varias flags repartidas por distintos servicios.

## Reconocimiento

```
nmap --privileged -sS -n -vv -p22,80,139,445,8081,10001,10121 -sV -sC -oN scan.txt 10.128.179.222
```

Puertos abiertos: SSH, HTTP, SMB (139/445) y tres puertos altos poco habituales (8081, 10001, 10121). El propio banner de SSH devuelve una flag embebida:

```
SSH-2.0-OpenSSH_8.2p1 THM{946219583339...}
```

## FTP/SMB

Entre los recursos accesibles había un usuario `quinn` y otro `eddie` (`users.txt`), y en un share de acceso anónimo una segunda flag:

```
THM{QUINN_IS_BACK007}
```

## Resto de servicios

Cada uno de los puertos altos escondía su propia pista o flag siguiendo el mismo patrón (banner o archivo accesible sin autenticación), típico de una sala pensada para practicar enumeración de servicios más que explotación de una vulnerabilidad concreta.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
