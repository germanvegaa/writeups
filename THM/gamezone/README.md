# GameZone - TryHackMe

**Fecha de resolución:** 29 de agosto de 2026

Máquina Linux, IP 10.129.159.37. Portal de videojuegos.

## Enumeración web

`index.php` incluye un buscador que envía peticiones POST a `portal.php` con el parámetro `searchitem`. Capturé la petición con Burp (`burp.txt`, `request.txt`) para trabajar sobre ella.

## Explotación

El parámetro `searchitem` de `portal.php` es inyectable. Pasando la petición capturada a sqlmap:

```
sqlmap -r request.txt --dbms=mysql --dump
```

sqlmap identifica la inyección y vuelca la base de datos, con un hash de contraseña (`hash.txt`) asociado a un usuario del sistema. Lo crackeé offline y usé esas credenciales para entrar por SSH.

## Post-explotación

Una vez dentro, una revisión de puertos internos (`netstat`) muestra un servicio adicional escuchando solo en localhost. Levanté un túnel SSH hacia ese puerto para poder acceder a él desde mi propia máquina.

## Escalada a root

El servicio interno resultó vulnerable a un exploit público con módulo ya disponible en Metasploit, que entrega ejecución directa como root al dispararlo a través del túnel.

---

GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
