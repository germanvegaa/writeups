# Brainstorm - TryHackMe

**Fecha de resolución:** 4 de septiembre de 2026

Sala de buffer overflow en Windows. El objetivo es `chatserver.exe`, un servidor de chat que escucha en el puerto 9999 y usa la librería `essfunc.dll`.

## Análisis del binario

`chatserver.exe` pide un nombre de usuario y después un mensaje, que devuelve por pantalla junto con la fecha. `essfunc.dll` está compilada sin ninguna protección de memoria (sin ASLR, sin DEP, sin canarios), lo que la convierte en el punto de apoyo perfecto para el exploit: cualquier instrucción `JMP ESP` dentro de ella sirve para redirigir la ejecución sin preocuparse por protecciones.

## Desarrollo del exploit

Metodología clásica de stack overflow: fuzzing para encontrar el punto de caída, cálculo del offset exacto hasta EIP, comprobación de bad characters con el set completo de bytes, y localización de un `JMP ESP` dentro de `essfunc.dll` para usar como dirección de retorno.

Con eso montado, generé el shellcode (reverse shell) evitando los bad characters detectados, con un pequeño NOP sled antes del shellcode para absorber el salto. Al enviar el mensaje al chatserver, el proceso salta a `essfunc.dll`, cae en el `JMP ESP` y ejecuta el shellcode directamente en el stack.

## Resultado

`chatserver.exe` corre como SYSTEM, así que no hace falta ninguna escalada adicional: el mismo overflow entrega una shell con privilegios máximos.

---

GitHub: [github.com/germanvegaa](https://github.com/germanvegaa)
TryHackMe: [tryhackme.com/p/G3RM4N](https://tryhackme.com/p/G3RM4N)
HackTheBox: [app.hackthebox.com/public/users/3990132](https://app.hackthebox.com/public/users/3990132)
