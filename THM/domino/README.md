# Domino - TryHackMe

**Fecha de resolución:** 25 de agosto de 2026

Máquina orientada a "Nexus Corp", con una aplicación Node (`app.js`) como servicio principal.

## Enumeración

De la propia aplicación o de una filtración asociada obtuve el listado de empleados de la empresa:

```
sarah.johnson@nexus.corp
michael.chen@nexus.corp
laura.hayes@nexus.corp
robert.wilson@nexus.corp
emma.taylor@nexus.corp
david.brown@nexus.corp
james.wright@nexus.corp
```

útil tanto para enumerar cuentas válidas como para dar formato a payloads con datos reales de la organización.

## Explotación

Antes de lanzar nada más elaborado, subí un `test.php` mínimo para confirmar que el servidor ejecutaba PHP subido por mí:

```php
<?php
echo "¡Hola, mundo!";
?>
```

Confirmada la ejecución, subí `php-reverse-shell.php` (pentestmonkey) por el mismo vector y obtuve conexión a mi listener, quedando con ejecución de código en el servidor.

---

GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
