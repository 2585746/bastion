# Runbook 02 — Cerrar la infraestructura (fase A, pasos A2–A7)

> **Objetivo:** dejar la base terminada antes de construir encima: copias que incluyen todo, memoria suficiente en docker01, decisión sobre Paperless, SSH endurecido en pve01, alertas limpias y el proyecto en GitHub.
> **Antes:** haz el runbook 01 (copia a Drive).
> **Dónde:** se indica en cada paso (pve01, docker01 o tu PC).
> **Regla:** un paso, una comprobación. No sigas si la salida no es la esperada.

---

## A2 — Copias en "Modo de selección: Todo"

**Por qué:** hoy la tarea lista las máquinas una a una (`100,101,103`). Una máquina nueva no entraría y se repetiría la incidencia 24.

1. Proxmox → **Centro de datos → Respaldo** → selecciona la tarea de las 03:30 → **Editar**.
2. **Modo de selección:** `Todo`. Guarda.
3. Compruébalo en pve01:

```
cat /etc/pve/jobs.cfg
```

`cat` muestra el archivo donde Proxmox guarda las tareas programadas. Debe aparecer la línea `all 1` y **no** debe aparecer `vmid 100,101,103`.

**Verificación real:** en la notificación de la próxima copia (03:30), la orden debe decir `--all 1`.

---

## A3 — Subir docker01 a 4 GB

**Por qué:** está al 89 % de memoria con 3 GB. El anfitrión tiene unos 10 GB libres, así que no hace falta esperar al hardware nuevo.

En **pve01**:

```
qm set 103 --memory 4096 --balloon 2048
```

`qm set` cambia la configuración de una VM. `--memory 4096` es el máximo (4 GB) y `--balloon 2048` el mínimo garantizado: si el anfitrión necesita memoria, puede recuperar hasta dejarla en 2 GB.

El cambio de memoria máxima se aplica al apagar y encender la VM (no vale reiniciar desde dentro):

```
qm shutdown 103 && sleep 20 && qm start 103
```

`qm shutdown` apaga de forma ordenada, `sleep 20` espera 20 segundos y `qm start` la enciende. Durante un minuto no habrá Homepage, n8n ni proxy: Kuma avisará por ntfy y es normal.

Comprueba dentro de **docker01**:

```
free -h
```

`free -h` muestra la memoria en unidades legibles. La columna `total` debe rondar 3,8 Gi.

```
docker ps --format 'table {{.Names}}\t{{.Status}}'
```

Lista los contenedores con su estado. Todos deben volver a `Up` (GLPI no, está parado a propósito).

**Inventario:** sección 5, tabla de máquinas, docker01 → `4 GB con ballooning (mín. 2 GB)`.

---

## A4 — Paperless: conectarlo a Drive o apagarlo

Elige **una** de las dos opciones.

### Opción 1 — Integrarlo con Google Drive

**Idea:** guardas o escaneas desde el móvil en una carpeta de Drive (`Paperless-Entrada`). Cada 5 minutos, rclone **mueve** lo que haya ahí a la carpeta de entrada (*consume*) de Paperless, que le hace OCR, lo clasifica y lo deja buscable.

**1. Localiza la carpeta consume.** En docker01:

```
grep -n consume /opt/paperless/docker-compose.yml
```

`grep -n` busca la palabra y muestra el número de línea. Verás algo como `./consume:/usr/src/paperless/consume`: la parte izquierda es la carpeta real (`/opt/paperless/consume`).

**2. Instala rclone en docker01:**

```
sudo apt update && sudo apt install -y rclone
```

**3. Configura el remoto de Drive.** Igual que en el runbook 01, paso 3, pero aquí como usuario `martin` y **sin** remoto de cifrado (Paperless necesita leer los archivos):

```
rclone config
```

Nombre del remoto: `gdrive`. Puedes reutilizar el mismo *client ID* del runbook 01.

**4. Crea en Drive la carpeta** `Paperless-Entrada` y sube un PDF de prueba.

**5. Prueba a mano:**

```
rclone move gdrive:Paperless-Entrada /opt/paperless/consume --min-age 1m -v
```

`rclone move` copia y después borra el original en Drive. `--min-age 1m` ignora archivos con menos de un minuto, para no coger uno que todavía se está subiendo. `-v` muestra lo que hace. En un minuto, el documento debe aparecer en Paperless.

**6. Automatízalo** con un temporizador de systemd:

```
sudo nano /etc/systemd/system/paperless-drive.service
```

```ini
[Unit]
Description=Mover documentos de Drive a Paperless
After=network-online.target

[Service]
Type=oneshot
User=martin
ExecStart=/usr/bin/rclone move gdrive:Paperless-Entrada /opt/paperless/consume --min-age 1m
```

`User=martin` hace que se ejecute con tu usuario, que es quien tiene la configuración de rclone y permisos en la carpeta.

```
sudo nano /etc/systemd/system/paperless-drive.timer
```

```ini
[Unit]
Description=Revisar Drive cada 5 minutos

[Timer]
OnBootSec=2min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
```

```
sudo systemctl daemon-reload && sudo systemctl enable --now paperless-drive.timer
```

Recarga systemd y activa el temporizador ahora y en cada arranque. Compruébalo:

```
systemctl list-timers paperless-drive.timer
```

**Aviso de privacidad:** el token de rclone en docker01 da acceso a tu Drive. Queda en `~/.config/rclone/rclone.conf` con permisos 600 y entra en la copia de la VM, que va cifrada a Drive.

### Opción 2 — Apagarlo

```
cd /opt/paperless && docker compose stop
```

Para los contenedores sin borrar nada (datos y configuración se conservan). Después: pausa el monitor `paperless OK` en Uptime Kuma y quita la tarjeta de `services.yaml` en Homepage. Para volver: `docker compose start`.

**Inventario:** sección 6, Paperless, fila "Estado", con lo que hayas elegido.

---

## A5 — SSH de pve01 solo con clave

> ⚠️ **Hazlo en casa**, con la consola web de Proxmox abierta en otra pestaña como vía de emergencia. Es la regla: no se toca el acceso de un servidor sin plan B.

**Por qué:** hoy pve01 acepta la contraseña de root por SSH. Con clave, un atacante en tu red no puede probar contraseñas.

**1. Autoriza la clave de CADA equipo desde el que entras** (PC de casa, portátil, Termius en el móvil). Desde la PowerShell del PC (cuenta `Martin_`):

```
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh root@192.168.1.242 "cat >> /root/.ssh/authorized_keys"
```

`type` muestra tu clave **pública** y la tubería (`|`) la envía por SSH; `cat >>` la añade al final del archivo de claves autorizadas de root. Te pedirá la contraseña de root por última vez. Repite desde el portátil. En Termius, exporta la clave pública de la app y añádela igual (o pégala con `nano /root/.ssh/authorized_keys` desde una sesión abierta).

**2. Comprueba que entras SIN contraseña** desde cada equipo:

```
ssh root@192.168.1.242
```

Si te pide contraseña, la clave no está bien puesta: **no sigas**.

**3. Crea el archivo de endurecimiento** (en pve01):

```
nano /etc/ssh/sshd_config.d/01-hardening.conf
```

```
PermitRootLogin prohibit-password
PasswordAuthentication no
KbdInteractiveAuthentication no
```

- `PermitRootLogin prohibit-password`: root puede entrar, pero solo con clave. (Proxmox necesita que root pueda usar SSH para tareas internas, por eso no se pone `no`.)
- `PasswordAuthentication no` y `KbdInteractiveAuthentication no`: nadie entra con contraseña.
- El nombre empieza por `01-` porque en esa carpeta gana el primer archivo que define un valor (incidencia 7).

**4. Valida y aplica sin cerrar tu sesión:**

```
sshd -t && systemctl reload ssh
```

`sshd -t` revisa la sintaxis; solo si es correcta (`&&`) se recarga el servicio. `reload` no corta las sesiones abiertas.

**5. Verifica la configuración efectiva** (regla 3):

```
sshd -T | grep -E 'permitrootlogin|passwordauthentication'
```

Debe mostrar `permitrootlogin without-password` (es el nombre interno de `prohibit-password`) y `passwordauthentication no`.

**6. Prueba desde otra ventana**, sin cerrar la actual: debe entrar con clave. Y prueba que la contraseña ya no vale:

```
ssh -o PubkeyAuthentication=no root@192.168.1.242
```

`-o PubkeyAuthentication=no` obliga a intentar sin clave. Debe responder `Permission denied (publickey)`.

**Si algo falla:** entra por la consola web (Proxmox → pve01 → Shell), borra el archivo `01-hardening.conf` y ejecuta `systemctl reload ssh`.

**Inventario:** sección 5, fila "SSH" → `Solo clave (PermitRootLogin prohibit-password, PasswordAuthentication no). Claves: PC, portátil, móvil`.

---

## A6 — Alertas limpias

### Quitar `mail-to-root`

Nunca ha entregado un correo (Gmail lo rechaza). Un canal de alerta que no funciona da falsa seguridad.

Proxmox → **Centro de datos → Notificaciones** → pestaña de *matchers* → edita `default-matcher` → en "Destinos a notificar" deja marcado **solo `ntfy`**. Guarda y pulsa **Test** en el destino `ntfy`: debe llegarte al móvil.

### Cambiar el tema de ntfy

El tema actual apareció en capturas de pantalla, y en ntfy.sh quien conoce el tema puede leer y enviar mensajes.

**1. Genera un nombre nuevo imposible de adivinar** (en cualquier Linux):

```
echo "bastion-$(openssl rand -hex 12)"
```

`openssl rand -hex 12` genera 24 caracteres aleatorios. Guarda el resultado en el gestor de contraseñas. **No lo pegues en chats ni en GitHub.**

**2. Cámbialo en todos los sitios que publican**, y prueba cada uno:

| Dónde | Cómo |
|---|---|
| App ntfy del móvil | Suscríbete al tema nuevo (no borres el viejo hasta terminar) |
| Proxmox | Centro de datos → Notificaciones → destino `ntfy` → editar el secreto `topic` → Test |
| Uptime Kuma | Ajustes → Notificaciones → editar la de ntfy → campo del tema → Probar |
| Healthchecks.io | Integrations → ntfy → cambiar el tema → Send test |
| n8n | Nodo `ntfy - Enviar` del informe semanal (URL) → ejecutar el flujo |
| pve01 | `nano /usr/local/bin/backup-to-drive.sh` → variable `NTFY_URL` → ejecutar el script |

**3.** Cuando los seis lleguen al tema nuevo, borra la suscripción al viejo.

---

## A7 — Subir Bastión a GitHub

**1. Renombra el repositorio.** En GitHub → tu repositorio → **Settings → General → Repository name** → `bastion`. GitHub redirige los enlaces antiguos.

**2. Clónalo en tu PC** (PowerShell, en la carpeta donde guardes proyectos):

```
git clone https://github.com/2585746/bastion.git
```

`git clone` descarga el repositorio con su historial.

**3. Copia dentro el contenido de `bastion-repo.zip`** (README, `docs/`, `runbooks/`, `.gitignore`). Mueve los README antiguos a `docs/memorias/` si quieres conservarlos.

**4. Busca secretos antes de subir nada** (esto es lo importante):

```
git grep -nIE "ntfy\.sh/[A-Za-z0-9_-]+|PVEAPIToken=|BEGIN OPENSSH|password\s*[:=]" -- . ; git status
```

`git grep` busca patrones en los archivos: temas de ntfy, tokens de Proxmox, claves privadas y contraseñas. Si sale algo que no sea un ejemplo (`TU-TEMA-NTFY`), bórralo del archivo antes de continuar. `git status` lista lo que se va a subir.

**5. Sube:**

```
git add -A
git commit -m "Reestructura el proyecto como Bastion: estado actual, plan maestro y runbooks"
git push
```

`git add -A` prepara todos los cambios, `git commit` los guarda con un mensaje y `git push` los envía a GitHub.

**6. A partir de ahora:** cada fase cerrada = marcar la casilla en `docs/ROADMAP.md`, actualizar el inventario y hacer un *commit*. El historial es tu prueba de progreso.

> Si alguna vez subes un secreto por error: **cámbialo** (rótalo) de inmediato. Borrarlo del archivo no basta, porque queda en el historial.

---

## Verificación final de la fase A

- [ ] `jobs.cfg` muestra `all 1`
- [ ] docker01 con ~4 GB y todos los contenedores `Up`
- [ ] Paperless integrado con Drive o apagado y documentado
- [ ] pve01 rechaza contraseñas por SSH (`Permission denied (publickey)`)
- [ ] Solo `ntfy` como destino de notificaciones, con tema nuevo probado en los seis sitios
- [ ] Repositorio `bastion` en GitHub sin secretos
- [ ] Inventario y ROADMAP actualizados
