# Runbook 02 — Cerrar la infraestructura (fase A, pasos A2–A7)

> **Objetivo:** dejar la base terminada antes de construir encima: copias que incluyen todo, memoria suficiente en docker01, decisión sobre Paperless, SSH endurecido en pve01, alertas limpias y el proyecto en GitHub.
> **Antes:** haz el runbook 01 (copia a Drive).
> **Dónde:** se indica en cada paso (pve01, docker01 o tu PC).
> **Regla:** un paso, una comprobación. No sigas si la salida no es la esperada.

---

## A2 — Copias en "Modo de selección: Todo" (✅ Completado)

**Por qué:** antes la tarea listaba las máquinas una a una (`100,101,103`). Una máquina nueva no entraría y se repetiría la incidencia 24.

1. Proxmox → **Centro de datos → Respaldo** → selecciona la tarea de las 03:30 → **Editar** (o editar `/etc/pve/jobs.cfg` en `pve01`).
2. **Modo de selección:** `Todo` (`all 1`). Guarda.
3. Compruébalo en pve01:

```
cat /etc/pve/jobs.cfg
```

`cat` muestra el archivo donde Proxmox guarda las tareas programadas. Aparece la línea `all 1` y ya **no** aparece `vmid 100,101,103`.

---

## A3 — Subir docker01 a 4 GB (✅ Completado)

**Por qué:** estaba al 89 % de memoria con 3 GB. El anfitrión tiene unos 10 GB libres, así que no hace falta esperar al hardware nuevo.

En **pve01**:

```
qm set 103 --memory 4096 --balloon 2048
```

`qm set` cambia la configuración de una VM. `--memory 4096` es el máximo (4 GB) y `--balloon 2048` el mínimo garantizado: si el anfitrión necesita memoria, puede recuperar hasta dejarla en 2 GB.

El cambio de memoria máxima se aplica al apagar y encender la VM (no vale reiniciar desde dentro):

```
qm shutdown 103 && sleep 20 && qm start 103
```

Comprueba dentro de **docker01**:

```
free -h
```

```
docker ps --format 'table {{.Names}}\t{{.Status}}'
```

---

## A4 — Paperless: conectarlo a Drive o apagarlo (✅ Completado: Opción 2)

Se eligió la **Opción 2 — Apagarlo**:

```
cd /opt/paperless && docker compose stop
```

Para los contenedores sin borrar nada (datos y configuración se conservan). Se pausó el monitor `paperless OK` en Uptime Kuma (y se sacó de la página de estado `homelab`) y se comentaron todas las líneas del bloque de Paperless en `/opt/homepage/config/services.yaml`. Para volver a encenderlo en el futuro: `docker compose start`.

---

## A5 — SSH de pve01 solo con clave (⏹️ Descartado por decisión el 13/11/2026; se conserva como referencia)

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

## A6 — Alertas limpias (✅ Completado)

### Quitar `mail-to-root`

Nunca ha entregado un correo (Gmail lo rechaza). Un canal de alerta que no funciona da falsa seguridad.

Proxmox → **Centro de datos → Notificaciones** → pestaña de *matchers* → edita `default-matcher` → en "Destinos a notificar" se dejó marcado **solo `ntfy`**.

*(Nota: sobre el cambio del tema de ntfy, se decidió conservar el tema actual).*

---

## A7 — Subir Bastión a GitHub (✅ Completado el 13/11/2026)

**1. Renombra el repositorio.** En GitHub → tu repositorio → **Settings → General → Repository name** → `bastion`. GitHub redirige los enlaces antiguos.

**2. Clónalo en tu PC** (PowerShell, en la carpeta donde guardes proyectos):

```
git clone https://github.com/2585746/bastion.git
```

`git clone` descarga el repositorio con su historial.

**3. Copia dentro el contenido de `bastion-repo.zip`** (README, `docs/`, `runbooks/`, `.gitignore`). Mueve los README antiguos a `docs/memorias/` si quieres conservarlos.

**4. Busca secretos antes de subir nada** (esto es lo importante):

```
git grep -nIE "ntfy\.sh/[A-Za-z0-9_-]+|PVEAPIToken[=]|BEGIN[ ]OPENSSH|password\s*[:=]" -- . ; git status
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

- [x] `jobs.cfg` muestra `all 1`
- [x] docker01 con ~4 GB y todos los contenedores `Up`
- [x] Paperless apagado y documentado
- [ ] ~~pve01 rechaza contraseñas por SSH~~ — *descartado por decisión (A5)*
- [x] Solo `ntfy` como destino de notificaciones en Proxmox
- [x] Repositorio `bastion` en GitHub sin secretos
- [x] Inventario y ROADMAP actualizados
