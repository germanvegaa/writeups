# Overpass 2 Hacked - TryHackMe

**Fecha de resolución:** 1 de septiembre de 2026

Continuación de Overpass: aquí el escenario es una captura de tráfico (pcap) del ataque que sufrió el servidor, más acceso posterior a la máquina ya comprometida.

## Análisis del tráfico

Revisando el pcap en Wireshark aparece una sesión de netcat en texto claro donde el atacante original escala al usuario `james`, y justo después clona un backdoor SSH desde GitHub (`NinjaJc01/ssh-backdoor`) y lo deja escuchando en el puerto 2222 como mecanismo de persistencia.

También va dentro del pcap una copia de `/etc/shadow` (`shadow.txt`) con los hashes de varios usuarios del sistema.

## Cracking de hashes

Con John y una wordlist a medida (`fasttrack.txt`, con variantes de temporada/año típicas de políticas corporativas) crackeé varios de esos hashes, incluido el de `james`.

## Acceso

Con la contraseña de `james` y el backdoor SSH visto en el pcap, me conecté directamente por el puerto 2222 al servidor ya comprometido.

## Escalada a root

En el home de `james` hay un binario `.suid_bash` con el bit SUID puesto y propietario root, dejado ahí por el propio atacante original como vía de escalada. Basta con invocarlo indicando que no baje privilegios:

```
./.suid_bash -p
```

y se obtiene una shell como root.

---

- GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
- TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
- HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
