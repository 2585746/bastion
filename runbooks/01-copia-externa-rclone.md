# Runbook 01 — Copia de seguridad cifrada a Google Drive con rclone (✅ Completado)

> **Objetivo:** subir las copias de Proxmox a Google Drive, **cifradas en el servidor**, para completar la regla 3-2-1. Google solo verá archivos ilegibles; solo tú, con la contraseña de cifrado, puedes recuperarlos.
> **Dónde se ejecuta:** en **pve01** (la sesión SSH como root, o la consola web).
> **Tiempo:** 30–45 minutos. **Requisito:** cada comando en su bloque; antes de seguir, comprueba la salida.

---

## Qué vamos a montar, en una frase

Proxmox ya deja las copias en `/var/lib/vz/dump`. rclone (un programa que se instala en pve01, no un servicio externo) coge esos archivos, los **cifra** y los sube a una carpeta de tu Drive. Un temporizador lo ejecuta cada noche después de la copia.

```
Proxmox (03:30) → /var/lib/vz/dump → rclone crypt (cifra en pve01) → Google Drive
```

Dos "remotos" de rclone encadenados:
1. `gdrive` → la conexión con tu cuenta de Google (guarda un token, no tu contraseña).
2. `gcrypt` → una capa de cifrado **encima** de `gdrive`. Subes a `gcrypt` y rclone cifra y reenvía a `gdrive`.

---

## Paso 0 — Antes de empezar

En tu **Drive**, crea a mano una carpeta llamada `bastion-backups`. Ahí irán las copias cifradas.

Decide **dos contraseñas** nuevas y guárdalas **ya** en tu gestor de contraseñas, con una nota que diga "cifrado rclone Bastión":
- Contraseña de cifrado de archivos.
- Contraseña de cifrado de nombres (*salt*).

> ⚠️ **Sin estas contraseñas, las copias de Drive son irrecuperables.** No viven solo en el servidor: si pve01 se quema, las recuperas desde otro equipo solo si las tienes en el gestor. Esta es la pieza más importante del runbook.

---

## Paso 1 — Instalar rclone

```
apt update && apt install -y rclone
```

`apt` instala rclone desde los repositorios de Debian. Comprueba la versión:

```
rclone version
```

Debe mostrar una versión (1.6x o superior). Si dijera "command not found", algo falló en la instalación; páramelo y lo miramos.

---

## Paso 2 — Crear tu propio "client ID" en Google (recomendado)

rclone trae un identificador compartido por miles de personas y va lento por los límites de Google. Crear el tuyo (gratis) hace la subida más rápida y fiable.

1. Entra en `https://console.cloud.google.com` con tu cuenta.
2. **APIs y servicios → Biblioteca →** busca **Google Drive API → Habilitar**.
3. **Google Auth Platform → Información de la marca:** nombre de la app "rclone", tu correo, URLs de página principal y política de privacidad (`https://github.com/2585746`) y dominio autorizado (`github.com`). Guarda.
4. **Google Auth Platform → Público:** pulsa **Publicar app** para pasarla a producción (así el token no caduca a los 7 días).
5. **Google Auth Platform → Clientes → Crear cliente de OAuth → Aplicación de escritorio.** Nombre "rclone".
6. Copia el **ID de cliente** y el **secreto de cliente**. Guárdalos en el gestor; los pega rclone en el paso siguiente.

---

## Paso 3 — Configurar el remoto de Google Drive (`gdrive`)

Esto es interactivo. Lanza:

```
rclone config
```

Responde así (lo que no se nombra, deja el valor por defecto pulsando Enter):

| Pregunta | Respuesta |
|---|---|
| `n/s/q` | **n** (nuevo remoto) |
| name | `gdrive` |
| Storage | busca **drive** (Google Drive) y pon su número |
| client_id | pega tu ID del paso 2 (o Enter si lo saltaste) |
| client_secret | pega tu secreto (o Enter) |
| scope | **1** (acceso completo) |
| service_account_file | Enter (vacío) |
| Edit advanced config | **n** |
| Use web browser to authenticate | ⚠️ ver abajo |

**Sobre la autenticación (importante en un servidor sin navegador):** pve01 no tiene navegador, así que cuando pregunte "Use auto config?" responde **n**. rclone mostrará un comando `rclone authorize "drive" ...`. Ejecútalo **en tu PC** (donde sí hay navegador y rclone instalado), inicia sesión con Google, y te dará un código largo. Pega ese código de vuelta en la consola de pve01.

Al final:
- "Configure this as a Shared Drive?" → **n**.
- Confirma el remoto → **y**.

**No salgas de `rclone config` todavía**: seguimos con el remoto de cifrado.

---

## Paso 4 — Configurar el remoto cifrado (`gcrypt`)

Seguimos dentro de `rclone config`:

| Pregunta | Respuesta |
|---|---|
| `n/s/q` | **n** |
| name | `gcrypt` |
| Storage | busca **crypt** y pon su número |
| remote | `gdrive:bastion-backups` (el remoto de antes + la carpeta) |
| filename_encryption | **1** / **standard** (cifra también los nombres) |
| directory_name_encryption | **1** / **true** |
| Password | elige **y** (escribir la tuya) y pega la **contraseña de cifrado de archivos** del paso 0 |
| Salt (password2) | **y** y pega la **contraseña de nombres** |
| Edit advanced config | **n** |
| Confirmar | **y** |

Sal con **q**.

> Las contraseñas se guardan cifradas en `/root/.config/rclone/rclone.conf`. Aun así, tú las tienes en el gestor (paso 0), que es lo que te salva si pierdes el servidor.

---

## Paso 5 — Probar que sube y cifra

Primero una prueba pequeña, sin tocar las copias de verdad:

```
echo "prueba bastion $(date)" > /tmp/prueba.txt
```

Crea un archivo de prueba con la fecha. Súbelo al remoto cifrado:

```
rclone copy /tmp/prueba.txt gcrypt: -P
```

`rclone copy` sube; `gcrypt:` es el destino cifrado; `-P` muestra el progreso. Ahora comprueba que rclone lo ve descifrado:

```
rclone ls gcrypt:
```

Debe listar `prueba.txt`. Y mira cómo lo ve Google (cifrado):

```
rclone ls gdrive:bastion-backups
```

Aquí el nombre sale como una cadena ilegible. **Eso es exactamente lo que queremos:** Google guarda algo que no puede leer. Si abres Drive en el navegador, verás un archivo con nombre cifrado en `bastion-backups`.

Borra la prueba:

```
rclone delete gcrypt:prueba.txt && rm /tmp/prueba.txt
```

---

## Paso 6 — El script de sincronización

Vamos a crear un pequeño script que sube la carpeta de copias y avisa por ntfy. Ábrelo:

```
nano /usr/local/bin/backup-to-drive.sh
```

Pega esto (cambia `TU-TEMA-NTFY` por tu tema real; no lo guardes en GitHub):

```bash
#!/bin/bash
set -euo pipefail

ORIGEN="/var/lib/vz/dump"
DESTINO="gcrypt:"
NTFY_URL="https://ntfy.sh/<TU-TEMA-NTFY>"

# Sincroniza solo los archivos de copia, no logs ni notas
if rclone sync "$ORIGEN" "$DESTINO" \
     --include "*.zst" --include "*.vma.zst" \
     --transfers 2 --bwlimit 8M \
     --log-file /var/log/rclone-backup.log --log-level INFO; then
  MENSAJE="Copia a Drive OK: $(rclone size gcrypt: --json | grep -o '"count":[0-9]*')"
  curl -s -H "Title: Bastion - copia a Drive" -d "$MENSAJE" "$NTFY_URL" >/dev/null
else
  curl -s -H "Title: Bastion - FALLO copia a Drive" -H "Priority: high" \
       -d "Revisar /var/log/rclone-backup.log" "$NTFY_URL" >/dev/null
  exit 1
fi
```

Qué hace, por partes:
- `set -euo pipefail`: si algo falla, el script se detiene en vez de seguir como si nada.
- `rclone sync`: deja el destino igual que el origen (sube lo nuevo, borra en Drive lo que ya no esté en local, respetando tu retención).
- `--include "*.zst"`: solo los archivos de copia.
- `--bwlimit 8M`: limita la subida a 8 MB/s para no saturar tu internet de casa de día.
- Avisa por ntfy tanto si va bien como si falla.

Guarda (**Ctrl+O**, Enter) y sal (**Ctrl+X**). Hazlo ejecutable:

```
chmod +x /usr/local/bin/backup-to-drive.sh
```

Pruébalo a mano una vez:

```
/usr/local/bin/backup-to-drive.sh
```

Debe llegarte la notificación "Bastion - copia a Drive" al móvil.

---

## Paso 7 — Programarlo cada noche

La copia de Proxmox es a las 03:30 y la completa dura ~2 minutos; subimos a Drive a las **04:00**, con margen. Usamos un temporizador de systemd (más controlable que cron). Crea el servicio:

```
nano /etc/systemd/system/backup-to-drive.service
```

```ini
[Unit]
Description=Subir copias de Proxmox a Google Drive (cifradas)
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup-to-drive.sh
```

Y el temporizador:

```
nano /etc/systemd/system/backup-to-drive.timer
```

```ini
[Unit]
Description=Copia diaria a Drive a las 04:00

[Timer]
OnCalendar=*-*-* 04:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

`Persistent=true` hace que, si el servidor estaba apagado a las 04:00 (corte de luz), la subida se lance al volver. Actívalo:

```
systemctl daemon-reload && systemctl enable --now backup-to-drive.timer
```

Comprueba que quedó programado:

```
systemctl list-timers backup-to-drive.timer
```

Debe aparecer con la próxima ejecución a las 04:00.

---

## Paso 8 — La prueba que de verdad importa: restaurar

Una copia que no se ha restaurado no está probada (regla de la fase 3). Simula que recuperas una copia desde cero:

```
mkdir -p /tmp/restore-test && rclone copy gcrypt: /tmp/restore-test --include "*100*" -P
```

Descarga y **descifra** las copias de kuma01 (CT 100) a una carpeta temporal. Comprueba que el archivo está y pesa lo esperado:

```
ls -lh /tmp/restore-test
```

Si el `.zst` aparece con su tamaño normal (~605 MB), el ciclo completo funciona: cifra, sube, descarga y descifra. Limpia:

```
rm -rf /tmp/restore-test
```

---

## Verificación final (todo en verde)

- [x] `rclone ls gcrypt:` lista las copias con nombre legible.
- [x] `rclone ls gdrive:bastion-backups` las muestra con nombre cifrado.
- [x] Llega la notificación de ntfy al ejecutar el script.
- [x] `systemctl list-timers` muestra la subida a las 04:00.
- [x] La prueba de restauración descarga y descifra bien.
- [x] Las dos contraseñas de cifrado están en el gestor de contraseñas.
