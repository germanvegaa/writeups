# Injectics - TryHackMe

**Fecha de resolución:** 9 de septiembre de 2026

Máquina Linux, IP 10.128.184.246. Sala centrada en inyección SQL.

## Enumeración web

El sitio tiene un `login.php` que llama de forma asíncrona a `functions.php` para validar credenciales:

```
POST /functions.php HTTP/1.1
Host: 10.128.184.246
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest

username=admin&password=test&function=login
```

## Explotación

El parámetro `username` de esa petición es inyectable. Con la petición capturada y pasada a sqlmap conseguí confirmar el punto de inyección y volcar la base de datos, incluyendo la tabla de usuarios de la aplicación (`users.txt`), lo que dio credenciales válidas para el panel de login sin necesidad de crackear ningún hash (las credenciales dumpeadas ya sirven directamente para el `function=login`).

---

GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
