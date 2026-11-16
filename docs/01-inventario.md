# Bastión — Inventario técnico

> Fuente de verdad de equipos, direcciones, servicios y configuración del proyecto **Bastión** (antes "Homelab").
> **Aquí nunca van contraseñas, claves, tokens ni el nombre del tema de ntfy.** Las credenciales están en mi gestor de contraseñas.
> Última actualización: 14/11/2026. Estados: ✅ funcionando · 🔄 en curso · ⏸️ apagado a propósito · 📦 recibido, sin montar · 🔜 planificado/pendiente · 🗑️ a retirar/retirado · ⚠️ sin confirmar

---

## 1. Red

| Dato | Valor |
|---|---|
| Router | Movistar WiFi 7 — `192.168.1.1` |
| Red local | `192.168.1.0/24`, máscara 255.255.255.0 |
| Rango DHCP del router | `192.168.1.33` – `192.168.1.199` |
| DNS que reparte el router por DHCP | `80.58.61.250` y `80.58.61.254` (Movistar). **No modificado** (decisión) |
| ¿El router responde como DNS en 192.168.1.1? | **No** de forma fiable (causa de la incidencia 13) |
| Rango para IP fijas del laboratorio | `192.168.1.240` en adelante (fuera del DHCP) |
| Dominio de Active Directory | `lab.internal` |
| Dominio para nombres locales (Pi-hole) | `home.arpa` (reservado para redes domésticas) |

---

## 2. Tabla de direcciones IP

| IP | Nombre | Qué es | Dónde vive | Estado |
|---|---|---|---|---|
| 192.168.1.1 | — | Router Movistar | — | ✅ |
| 192.168.1.40 (DHCP) | ws-cli01 | Cliente Windows 11 Enterprise (evaluación) | Hyper-V, PC | 🗑️ se reconstruye en Proxmox |
| 192.168.1.240 | — | Libre (antes ubuntu-srv-01, retirado el 04/11/2026) | — | Libre |
| 192.168.1.241 | ws-dc01 | Controlador de dominio `lab.internal` + DNS | Hyper-V, PC | 🗑️ se reconstruye en Proxmox (misma IP prevista) |
| 192.168.1.242 | **pve01** | Anfitrión Proxmox VE | OptiPlex 7020 | ✅ |
| 192.168.1.243 | **kuma01** | Uptime Kuma (CT 100) | pve01 | ✅ |
| 192.168.1.244 | **pihole01** | Pi-hole (CT 101) | pve01 | ✅ |
| 192.168.1.245 | lab01 | Contenedor de prácticas de Linux (CT 102) | pve01 | ✅ |
| 192.168.1.246 | ansible01 | Nodo de control de Ansible | pve01 | 🔜 propuesta |
| 192.168.1.247 | rocky-srv-01 | Rocky Linux, miembro del dominio | pve01 | 🔜 propuesta |
| 192.168.1.248 | **docker01** | VM Debian 13 con Docker (NPM, Homepage, n8n, GLPI, Paperless ⏸️) | pve01 | ✅ |

IP libres para servicios nuevos: `.240` y de `.249` en adelante. **Actualizar esta tabla cada vez.**

---

## 3. Tailscale

| Dato | Valor |
|---|---|
| Tailnet | MagicDNS activo (nombre en el gestor de contraseñas; no se publica) |
| Router de subred | pve01 anuncia `192.168.1.0/24` (ruta aprobada). Reenvío IP en `/etc/sysctl.d/99-tailscale.conf` |
| DNS global de la tailnet | `192.168.1.244` (Pi-hole) con **Override DNS servers** activado |
| DNS en pve01 | `accept-dns=false`: pve01 **no** usa el DNS de la tailnet (usa 1.1.1.1). Ver incidencia 23 |
| Efecto conocido | El router de subred hace SNAT: en Pi-hole todas las consultas de la tailnet aparecen como `192.168.1.242` |

| Dispositivo | IP de Tailscale | Caducidad de clave | Notas |
|---|---|---|---|
| pve01 | ver gestor de contraseñas | Desactivada | Servidor |
| pixel-10a | ver gestor de contraseñas | Desactivada | Móvil, uso remoto principal (chat del agente, Homepage) |
| ubuntusrv | ver gestor de contraseñas | Desactivada | 🗑️ ⚠️ Confirmar que se quitó de la tailnet; si ya no está, borrar esta fila |

---

## 4. Hardware

### Servidor: Dell OptiPlex 7020 SFF (pve01)

> 📦 **Recibido y pendiente de montar** (sesión de hardware en Yepes): RAM 2×8 GB, HDD de 2 TB y el Kingston A400. En la misma sesión: pasta térmica, Memtest86+ (también para descartar la RAM como causa de las caídas) y SMART largo del HDD.

| Componente | Actual | Tras la sesión de hardware 🔜 |
|---|---|---|
| Procesador | Intel Core i5-4590 (4 núcleos / 4 hilos, Haswell), gráfica HD 4600 (Quick Sync H.264) | — |
| Memoria | 16 GB DDR3-1600: 4×4 GB (3 SK Hynix + 1 "Noname"). Uso típico del anfitrión: ~5–6 GB | **24 GB**: 2×8 GB Patriot Viper + 2×4 GB Hynix (quitar el Noname). Un módulo de 8 en cada canal |
| Disco 1 | **Toshiba TR200 240 GB** (SATA). SMART PASSED, sin errores, 26 588 h, desgaste 63, ~60 TB escritos. LVM: `root` 64 GB (`local`), `swap` 8 GB, `data` ~140 GB (`local-lvm`) | Proxmox, ISO, plantillas y contenedores ligeros. Sustituir a medio plazo |
| Disco 2 | — | **Kingston A400 240 GB**: salud 79 %, 3 465 h, 0 sectores malos, 27,9 TB escritos. → **LVM-Thin `vmdata`** para las VM |
| Disco 3 | — | **HDD Samsung HD203WI 2 TB** (5400 rpm, fabricado en 2010). → copias de seguridad + biblioteca de Jellyfin. Hacer SMART largo al montarlo |
| Lector DVD | Instalado | Se retira; su bahía aloja el Kingston con adaptador *caddy* |
| Red | 1 GbE integrada (Intel) | — |
| Fuente | Propietaria de Dell (~255 W) | — |
| Temperatura | En reposo 26–32 °C (Glances, `Package id 0`). ~69 °C en Memtest86+ | Cambiar pasta térmica |

**BIOS:** VT-x y VT-d activados · UEFI · Secure Boot desactivado · SATA en AHCI · AC Recovery = Power On · errores de teclado desactivados.

### PC de casa (Windows 11 Home 26H2)

| Componente | Valor |
|---|---|
| Placa | Gigabyte B550M K (BIOS F7) |
| Procesador | AMD Ryzen 7 5700X |
| Memoria | 16 GB |
| Gráfica | NVIDIA RTX 5060 (8 GB) — para Ollama/IA |
| C: (Windows) | **Samsung PM981 256 GB NVMe** (MZVLB256HBHQ-000L7), clonado el 02/11/2026. ~15 GB sin asignar al final |
| D: Juegos | SSD de 480 GB |
| Kingston A400 240 GB | Conectado pero **desactivado** por colisión de firmas. No poner en línea; va al servidor |
| AMD RX 6500 XT | Sin usar. No sirve para el servidor (no cabe, sin alimentación, sin codificador de vídeo). Pensado venderla |
| Cuentas de Windows | `Martin` (PowerShell normal, **sin clave SSH** 🔜) · `Martin_` (administrador, tiene la clave Ed25519 autorizada en docker01 y `rclone v1.75.2` instalado) |
| Terminal | **Warp**: sesiones SSH a pve01 (root) y docker01 (martin). Modo Terminal; sin sugerencias de IA en las clases |

### Portátil de Aranjuez

| Dato | Valor |
|---|---|
| Clave SSH | Ed25519 propia (`portatil-aranjuez`), autorizada en docker01 |

---

## 5. Proxmox VE (pve01)

| Dato | Valor |
|---|---|
| Versión | Proxmox VE 9.2.x, núcleo `7.0.14-26-pve` (Debian 13.7 "trixie") |
| Instalación | ext4 en el TR200 (`/dev/sda`). Almacenamientos: `local` (en `root`, 64 GB) y `local-lvm` (`data`, ~140 GB) |
| Repositorios | Enterprise desactivados · **pve-no-subscription** activado |
| DNS | Principal `1.1.1.1`, secundario `192.168.1.1`, dominio de búsqueda `lab.internal`. **Nunca** un DNS alojado en el propio pve01 |
| Usuarios | `martin@pve` (usuario **de Proxmox**, Administrator en `/`, uso diario) · `root@pam` (emergencias). Ambos con **2FA TOTP** (Aegis) y códigos de recuperación |
| Tokens de API | **`martin@pve!homepage`** (*Privilege Separation*, `PVEAuditor` en `/`, solo lectura, lo usa Homepage) · **`martin@pve!n8n`** (*Privilege Separation*, `PVEAuditor` en `/`, solo lectura, lo usan el informe semanal y el agente) |
| SSH | Solo existe el usuario Linux `root`. Acepta **contraseña de root** (⏹️ A5 descartado por decisión el 13/11/2026: se mantiene el acceso por contraseña). La clave del PC no está autorizada para root. Termius en el móvil |
| Consola web | `~/.inputrc` con `set enable-bracketed-paste off`. noVNC no permite pegar: usar Tab para autocompletar |
| Notificaciones | Destino **`ntfy`** (webhook POST, URL `https://ntfy.sh/{{ secrets.topic }}`, cabecera `Title: {{ title }}`, cuerpo `{{ message }}`) — **único aviso real**. Los avisos de más de 4 KB llegan como adjunto `attachment.txt`. `default-matcher` → **solo `ntfy`** (`mail-to-root` desvinculado en Fase A) ✅ |
| Vigilancia externa | Healthchecks.io, chequeo `pve01`, periodo 5 min, gracia 5 min, integración ntfy. Cron en `/etc/cron.d/healthchecks` |
| Hardware | **Glances 4.3.1** (apt) como `glances-web.service` (`Restart=always`, `RestartSec=10`). Orden: `glances -w --disable-webui -B 192.168.1.242` → solo API en `192.168.1.242:61208` (`/api/4/...`). Sensor de CPU: `Package id 0` (aviso 80 °C, crítico 100 °C) ✅ |
| Estabilidad | Varias caídas sin apagado ordenado (30/10, 01/11 y tres el 04/11), probablemente cortes de luz. AC Recovery = Power On las recupera. Tras un reinicio, comprobar que **todo** volvió |
| Copias de seguridad | Tarea diaria **03:30**, modo snapshot, ZSTD, destino `local`. ✅ **Modo de selección: Todo** (`all 1` en `/etc/pve/jobs.cfg`). Retención **temporal 3 diarias + 1 semanal** (copia completa ≈ 6 GB: kuma01 0,6 · pihole01 0,3 · docker01 5,0; ~18–24 GB en total); volver a 7+2 (~54 GB) cuando estén en el HDD. Restauración probada, incluida una VM con base de datos |
| Copia fuera de casa | ✅ `rclone v1.60.1-DEV` con `crypt` (`gdrive` → `gcrypt`) hacia `Drive/bastion-backups` con Client ID OAuth propio en producción; cifrado de archivos y nombres en `pve01`; script `/usr/local/bin/backup-to-drive.sh` y temporizador `backup-to-drive.timer` a las **04:00**; aviso por `ntfy`; restauración probada |
| Salud de discos | smartd activo; avisos vía notificaciones |

### Contenedores y máquinas virtuales

| ID | Nombre | Tipo | SO | Recursos | IP | Estado |
|---|---|---|---|---|---|---|
| 100 | kuma01 | LXC sin privilegios (nesting=1), `onboot=1` | Debian 13 | 1 núcleo · 512 MB · swap 512 · 4 GB | .243 | ✅ |
| 101 | pihole01 | LXC sin privilegios, `onboot=1` | Debian 13 | 1 núcleo · 512 MB · swap 512 · 4 GB | .244 | ✅ |
| 102 | lab01 | LXC sin privilegios | Debian 13 | 1 núcleo · 512 MB · 4 GB | .245 | ✅ Creado y arrancado el 13/11/2026, con *nesting* activado · ⚠️ confirmar instantánea `limpio` |
| 103 | docker01 | VM, `onboot=1` | Debian 13 | 2 vCPU (`host`) · **4 GB con ballooning (mín. 2 GB)** · 20 GB | .248 | ✅ |
| — | ansible01 | LXC | Debian 13 | ligero | .246 | 🔜 |
| — | rocky-srv-01 | VM | Rocky Linux | ~1,5 GB | .247 | 🔜 |
| — | ws-dc01 | VM | Windows Server (VirtIO) | 1,5–3 GB | .241 | 🔜 reconstrucción |
| — | ws-cli01 | VM | Windows 11 (vTPM, saltar requisitos CPU) | 4 GB | DHCP | 🔜 reconstrucción |

Plantilla descargada: `debian-13-standard_13.1-2_amd64.tar.zst`. Todos los contenedores usan DNS `1.1.1.1` y dominio `lab.internal`.

### Detalle de docker01 (VM 103)

| Dato | Valor |
|---|---|
| Hardware virtual | 2 vCPU tipo `host`, **4 GB con ballooning (mín. 2 GB)**, disco 20 GB VirtIO SCSI en `local-lvm` (*discard* + *SSD emulation*), red VirtIO, QEMU Guest Agent |
| Sistema | Debian 13 netinst (solo SSH y utilidades estándar) |
| Red | IP fija en `/etc/network/interfaces` (`ens18`), DNS `1.1.1.1` / `1.0.0.1` en `/etc/resolv.conf` (no usa Pi-hole) |
| SSH | Usuario `martin` con `sudo` y en el grupo `docker`. Solo clave Ed25519; `PermitRootLogin no` y `PasswordAuthentication no` en `sshd_config.d/01-hardening.conf`. Claves autorizadas: PC de casa (cuenta `Martin_`) y portátil de Aranjuez. ⚠️ Confirmar `sudo sshd -T \| grep passwordauthentication` → `no` |
| Docker | Docker Engine + Compose v5.6.1 desde el repositorio oficial de Docker |
| Organización | Un directorio por servicio: `/opt/<servicio>/compose.yaml`. Secretos en `.env` con permisos 600 |
| Puertos usados | 80 / 81 / 443 (NPM) · 3000 (Homepage) · 5678 (n8n) · 8000 (Paperless ⏸️) · 8080 (GLPI) |
| Instantáneas | `base`, `npm-ok`, `glpi-ok`, `paperless-ok`, `n8n-ok` ⚠️ (si se hizo), `n8n-informe-ok` ⚠️ (opcional), `antes-glpi-reactivar` y `antes-importar-incidencias` (14/11/2026) |

---

## 6. Servicios

### Uptime Kuma — kuma01

| Dato | Valor |
|---|---|
| URL | `http://192.168.1.243:3001` · `http://kuma.home.arpa` |
| Versión | 2.5.8, instalada en `/opt/uptime-kuma` con Node.js de Debian |
| Servicio | `uptime-kuma.service` (systemd, `Restart=on-failure`) |
| Base de datos | SQLite (`/opt/uptime-kuma/data/kuma.db`) |
| Usuario | `administrador` (contraseña en el gestor) |
| Avisos | ntfy, prioridad alta, aplicado a todos los monitores |
| Página de estado | **`homelab`** (pública en la red local). La usan Homepage y n8n: `/api/status-page/homelab` (monitores) y `/api/status-page/heartbeat/homelab` (latidos y disponibilidad 24 h) |
| Restablecer contraseña | `cd /opt/uptime-kuma && npm run reset-password` |

Monitores activos (ID · nombre): 1 `pve01 - Web` · 2 `pve01-ping` · 3 `Internet Casa` · 6 `pihole01 - DNS` · 7 `docker01 - Ping` · 8 `npm - Panel` · 11 `Homepage Ping` · 12 `GLANCES RENDIMIENTO` · 13 `N8N HTTP`.
Monitores en pausa (fuera de la página de estado): 10 `paperless OK` ⏸️.
⚠️ 5 `glpi - Web`: GLPI está encendido desde el 14/11/2026; reactivar este monitor y confirmar que pasa a verde.
⚠️ Confirmar si existen todavía `ws-dc01 - DNS` y `ubuntu-srv-01` (no aparecen en la página de estado). Si existen, pausar el primero y borrar el segundo.

### Pi-hole — pihole01

| Dato | Valor |
|---|---|
| URL | `http://192.168.1.244/admin` · `http://pihole.home.arpa/admin` |
| Versión | Pi-hole v6 |
| Uso | Solo DNS de la tailnet de Tailscale (no de toda la casa) |
| DNS ascendente | Cloudflare (1.1.1.1) |
| Listas | StevenBlack (por defecto) + **Hagezi Pro** (formato adblock, jsDelivr). ~303 000 dominios |
| Modo de privacidad | 0 (mostrar todo) |
| NTP de FTL | ✅ desactivado (`ntp.sync.active false`). Un LXC sin privilegios no puede cambiar la hora (incidencia 18) |
| Registros locales | ✅ `pve.home.arpa` → .242 (directo, sin proxy) · `kuma`, `pihole`, `npm`, `glpi`, `paperless`, `homepage`, `n8n` `.home.arpa` → .248 (NPM) |
| *App password* | Una propia para el widget de Homepage |
| Reenvío condicional | 🔜 `lab.internal` → DC (.241) cuando exista el nuevo DC |

### Nginx Proxy Manager — docker01

| Dato | Valor |
|---|---|
| URL | Panel `http://192.168.1.248:81` · `http://npm.home.arpa` |
| Versión | 2.16 |
| Ubicación | `/opt/npm/compose.yaml` · configuración y certificados en volúmenes |
| Proxy hosts | `kuma.home.arpa` → .243:3001 · `pihole.home.arpa` → .244:80 · `paperless.home.arpa` → .248:8000 · `glpi.home.arpa` → .248:8080 · `homepage.home.arpa` → .248:3000 · `n8n.home.arpa` → .248:5678 · `npm.home.arpa` → 127.0.0.1:81 |
| Opciones | *Websockets* en Uptime Kuma, Paperless y n8n · *Block Common Exploits* activado · sin certificados (🔜 HTTPS con dominio propio) |
| Excepción | Proxmox **no** pasa por el proxy (ya va por HTTPS :8006) |

### Homepage — docker01

| Dato | Valor |
|---|---|
| URL | `http://homepage.home.arpa` · `http://192.168.1.248:3000` |
| Versión | 2.7.0 (`ghcr.io/gethomepage/homepage:latest`) |
| Ubicación | `/opt/homepage/compose.yaml` · configuración en `./config` (dueño `martin`, PUID/PGID 1000) |
| `HOMEPAGE_ALLOWED_HOSTS` | `homepage.home.arpa,192.168.1.248:3000` |
| Secretos | `/opt/homepage/.env` (permisos 600) con `HOMEPAGE_VAR_*` |
| Diseño (`settings.yaml`) | `title: Homelab` (🔜 cambiar a `Bastión`) · grupos Infraestructura y Servicios en filas de 3 · **Hardware** en fila de 4 |
| Widgets | Proxmox (token solo lectura) · Pi-hole v6 (*app password*) · Uptime Kuma (página de estado `homelab`) · recursos de docker01 · grupo **Hardware** de pve01 vía Glances (`sensor:Package id 0`, `cpu`, `memory`, `fs:/`) ✅ (bloque de Paperless comentado en `services.yaml`) |
| Enlaces | `href` con nombres `*.home.arpa`; `siteMonitor` con IP (docker01 no usa Pi-hole) |
| Estilo | Guía propia (oscuro + rojo `#E02B2F`): `color: red` + `config/custom.css` |
| Seguridad | **Sin socket de Docker montado** |

### n8n — docker01

| Dato | Valor |
|---|---|
| URL | `http://n8n.home.arpa` · `http://192.168.1.248:5678` |
| Versión | 2.46.6 (`docker.n8n.io/n8nio/n8n`) |
| Ubicación | `/opt/n8n/compose.yaml` · volumen `n8n_data` · `.env` (600) con `N8N_ENCRYPTION_KEY` |
| Variables | En `compose.yaml`: `N8N_HOST=n8n.home.arpa`, `N8N_WEBHOOK_URL=http://n8n.home.arpa/`, `N8N_SECURE_COOKIE=false`, `GENERIC_TIMEZONE` y `TZ` = `Europe/Madrid` |
| Credenciales (cifradas en n8n) | `Google Gemini(PaLM) Api` (clave del proyecto de AI Studio) · `Proxmox API (n8n)` (Header Auth, *Allowed domains*: 192.168.1.242) · Basic Auth del chat del agente |
| Vigilancia | Monitor `N8N HTTP` en Kuma (`/healthz`) |

**Flujos:**

| Flujo | Qué hace | Estado |
|---|---|---|
| Informe semanal Bastión | Lunes 9:00 · Kuma (monitores + disponibilidad 24 h) + Glances (memoria, sensores, disco) + Proxmox (tareas `vzdump` de 7 días) → nodo Code `Preparar datos` → **Gemini 3.6 Flash** → ntfy (`Title: Informe semanal homelab`, 🔜 cambiar a Bastión) | ✅ publicado; primera ejecución automática el 09/11 9:00 |
| Herramienta - Estado Bastión | Subflujo (*When Executed by Another Workflow*) con la misma cadena hasta `Preparar datos`; devuelve el texto `informe` | ✅ publicado |
| Agente Bastión | Chat público (*Hosted Chat*, **Basic Auth**) → AI Agent con **Gemini 3.8 Flash** + Simple Memory (10) · herramientas de solo lectura: `estado_general` (subflujo) y `maquinas_proxmox` (`/cluster/resources?type=vm`) | ✅ publicado · URL `http://n8n.home.arpa/webhook/<id>/chat` · acceso directo en el Pixel |

**API de Gemini (Google AI Studio):** proyecto "Proyecto de Gemini", nivel 1 · ~13 € de crédito · gasto real a 09/11: 0,09 € cubiertos por el descuento (≈0,01 € por llamada) · 🔜 límite de inversión mensual de **3 €** ⚠️ confirmar que se puso · "Logs and Datasets" **activado** (Google guarda peticiones y respuestas): **nada personal ni secretos a Gemini**. En este mismo proyecto vive el cliente OAuth de `rclone` (en estado de producción).

### Paperless-ngx — docker01 ⏸️

| Dato | Valor |
|---|---|
| URL | `http://paperless.home.arpa` · `http://192.168.1.248:8000` |
| Versión | 3.4.1 + PostgreSQL 18 + Valkey (Compose oficial, variante PostgreSQL) |
| Ubicación | `/opt/paperless/` · volúmenes con nombre gestionados por Docker |
| Configuración | OCR español + inglés · `Europe/Madrid` · `PAPERLESS_ALLOWED_HOSTS` y `PAPERLESS_CSRF_TRUSTED_ORIGINS` con nombre e IP:puerto · clave secreta en `docker-compose.env` con permisos 600 |
| Estado | ⏸️ **Apagado desde 13/11/2026** (`docker compose stop`), datos conservados e instantánea `paperless-ok`. Monitor en pausa y bloque comentado en `services.yaml` de Homepage |
| Encender | `cd /opt/paperless && docker compose start` |

### GLPI — docker01 ✅

| Dato | Valor |
|---|---|
| URL | `http://glpi.home.arpa` · `http://192.168.1.248:8080` |
| Versión | GLPI 12.0.1 (imagen oficial `glpi/glpi`) + MariaDB 11.4 (versión fijada) |
| Ubicación | `/opt/glpi/compose.yaml` · credenciales en `.env` (permisos 600) · `storage/glpi` con dueño 33:33 |
| Estado | ✅ **Encendido desde 14/11/2026** (estuvo apagado del 04/11 al 14/11). Fuente real del registro de incidencias: 27 importadas más la 29 |
| API | API legacy (`/apirest.php`) activada. Cliente de API `bastion` limitado a la IP interna de Docker (`172.19.0.1`), no a un rango abierto. Hacen falta los dos tokens: el de aplicación y el de usuario |
| Importación | `~/glpi-import/importar_incidencias.py` en `docker01`: lee `03-incidencias.md` y crea un ticket por incidencia (`--apply`; sin esa opción es una prueba en seco; `--only N` elige cuáles) |
| Homepage | ⚠️ Descomentar su bloque en `services.yaml` si se quiere en el panel |
| Encender | `cd /opt/glpi && docker compose start` |

---

## 7. Accesos rápidos (marcadores)

| Servicio | Enlace |
|---|---|
| Homepage (entrada principal) | `http://homepage.home.arpa` · `http://192.168.1.248:3000` |
| Agente Bastión (chat) | `http://n8n.home.arpa/webhook/<id>/chat` (Basic Auth) |
| Proxmox | `https://192.168.1.242:8006` · `https://pve.home.arpa:8006` |
| Proxmox (respaldo por IP de Tailscale) | `https://ver gestor de contraseñas:8006` |
| Uptime Kuma | `http://kuma.home.arpa` · `http://192.168.1.243:3001` |
| Pi-hole | `http://pihole.home.arpa/admin` · `http://192.168.1.244/admin` |
| Nginx Proxy Manager | `http://npm.home.arpa` · `http://192.168.1.248:81` |
| n8n (editor) | `http://n8n.home.arpa` · `http://192.168.1.248:5678` |
| Paperless-ngx ⏸️ | `http://paperless.home.arpa` · `http://192.168.1.248:8000` |
| GLPI ✅ | `http://glpi.home.arpa` · `http://192.168.1.248:8080` |
| Glances (API) | `http://192.168.1.242:61208/api/4/status` |
| Consola de Tailscale | `https://login.tailscale.com/admin` |
| Healthchecks | `https://healthchecks.io` |
| Google AI Studio (gasto de Gemini) | `https://aistudio.google.com` |

Fuera de casa funcionan con Tailscale activo en el móvil (router de subred). Los nombres `*.home.arpa` solo funcionan con Tailscale (Pi-hole); en casa sin Tailscale, usar la IP.

---

## 8. Dónde están los secretos (solo la ubicación)

| Secreto | Dónde |
|---|---|
| Contraseñas de `root@pam`, `martin@pve`, contenedores, Kuma, Pi-hole, NPM, Paperless, GLPI, n8n | Gestor de contraseñas |
| Códigos de recuperación 2FA de Proxmox | Gestor de contraseñas / copia en papel |
| Nombre del tema de ntfy | Gestor de contraseñas (en Proxmox está como *secret*; en n8n dentro de la URL del nodo ntfy; en `pve01` en `/usr/local/bin/backup-to-drive.sh`) |
| URL de ping de Healthchecks | Panel de Healthchecks y `/etc/cron.d/healthchecks` |
| Token de Proxmox para Homepage, *app password* de Pi-hole y token de Paperless | Gestor de contraseñas y `/opt/homepage/.env` (permisos 600) |
| Token de Proxmox para n8n (`martin@pve!n8n`) | Gestor de contraseñas y credencial cifrada en n8n |
| Clave de la API de Gemini | Gestor de contraseñas y credencial cifrada en n8n |
| Usuario y contraseña del chat del agente (Basic Auth) | Gestor de contraseñas y credencial cifrada en n8n |
| `N8N_ENCRYPTION_KEY` | `/opt/n8n/.env` (600) y gestor de contraseñas. Sin ella no se pueden descifrar las credenciales de n8n al restaurar |
| Token de aplicación (cliente de API `bastion`) y token de usuario de GLPI | Gestor de contraseñas. No se guardan en archivos; el script los lee de variables de entorno |
| Credenciales de bases de datos (GLPI, Paperless) | `.env` / archivo de configuración de cada servicio en `/opt/<servicio>/` (permisos 600) |
| Clave privada SSH | Solo en cada cliente (PC `Martin_`, portátil de Aranjuez y móvil). Nunca se copia entre equipos |
| Contraseñas de cifrado de rclone (archivos y nombres) y Client ID / Secret OAuth | Gestor de contraseñas (imprescindibles para restaurar las copias desde Google Drive si se pierde `pve01`) |
| Token de Google Drive y claves ofuscadas de rclone | `/root/.config/rclone/rclone.conf` en `pve01` (permisos 600) |
