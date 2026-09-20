# Intruder - TryHackMe

**Fecha de resolución:** 27 de agosto de 2026

Máquina Linux con un sitio corporativo de "MediaHub".

## Enumeración web

Revisando el código fuente y comentarios dejados en la aplicación apareció una nota de desarrollo que no debía estar en producción:

```
Developer Note (temporary)
--------------------------------------------------------------------------
Admin test account for staging environment
Email: admin@mediahub.thm

Password policy reminder:
Admin password follows company format:
MediaHub + any year

TODO: remove before production deployment
```

## Explotación

Con el formato de contraseña ya filtrado, probé `MediaHub2026` (año en curso) contra el login de administrador y entré directamente, sin necesidad de fuerza bruta.

## Post-explotación

Desde el panel de administración subí `php-reverse-shell.php` a través de la función de carga de archivos de la aplicación, y al acceder a ella obtuve ejecución de código en el servidor.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
