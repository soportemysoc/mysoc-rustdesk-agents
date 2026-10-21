# secret/ — clave de acceso permanente desatendido (RustDesk)

Este directorio contiene la **contraseña de acceso permanente** de RustDesk que se
aplica a la flota Windows de MYSOC como **acceso alternativo** (permite conectar y
cambiar de sesión con Windows bloqueado o en pantalla de inicio, junto con
`allow-logon-screen-password = 'Y'`).

> **El repositorio es público.** La contraseña **nunca** se publica en claro.

## `desatendido.enc`

Formato (una línea, ASCII):

```
MYSOC1.<ciphertext_b64>.<hmac_b64>
```

| Parámetro | Valor |
|---|---|
| Cifrado | AES-256-CBC + PKCS7 |
| Derivación | PBKDF2-HMAC-SHA256, 200 000 iteraciones, dklen 64 → `key`=dk[0:32], `mackey`=dk[32:64] |
| Salt | constante `MYSOC-rustdesk-desatendido-v1` |
| IV | constante `MYSOCrd-dsatd-v1` (16 bytes) |
| MAC | HMAC-SHA256(mackey, ciphertext) |

El cifrado es **determinista** (salt e IV fijos) a propósito: si la contraseña no
cambia, el archivo no cambia, y los equipos no reaplican nada. Si cambia, el blob
cambia y la tarea `MYSOC-RustDesk-Whitelist` lo aplica en la siguiente ejecución.

La clave (passphrase) **no está aquí**: vive en la bóveda GPG del agente
(`rustdesk-desatendido-enc-key`) y va embebida en el updater desplegado
(`ActualizarWhitelistRustDesk.ps1`) y en el instalador `InstalarRustDeskMYSOC`.

## Alcance de la protección

- **Protege contra:** cualquiera que lea el repositorio público (GitHub, buscadores,
  clones). Sin la passphrase el blob es inútil.
- **No protege contra:** quien obtenga el script desplegado (administrador local de un
  PC de la flota, o acceso a la carpeta `Scripts-PowerShell`), porque la passphrase va
  embebida en él. La contraseña es además **única para toda la flota**: un equipo
  comprometido revela la contraseña global. El control que acota el riesgo es
  `id-whitelist` en el endpoint: sin un ID de soporte autorizado, la contraseña sola no
  sirve.

## Rotación

```bash
python3 ~/.hermes/secrets/vault/vault.py set-pass rustdesk-desatendido-global   # nueva contraseña (stdin)
python3 ~/scripts/rustdesk-agents/publish_secret.py                             # cifra, verifica y publica
```

La tarea diaria (`publish-cron.sh`) republica automáticamente si detecta desincronía.
Rotar la **passphrase** (`rustdesk-desatendido-enc-key`) exige además redesplegar el
updater y el instalador en la flota.
