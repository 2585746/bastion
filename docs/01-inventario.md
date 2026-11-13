# BastiÃ³n â€” Inventario tÃ©cnico

> Fuente de verdad de equipos, direcciones, servicios y configuraciÃ³n del proyecto **BastiÃ³n** (antes "Homelab").
> **AquÃ­ nunca van contraseÃ±as, claves, tokens ni el nombre del tema de ntfy.** Las credenciales estÃ¡n en mi gestor de contraseÃ±as.
> Ãšltima actualizaciÃ³n: 09/11/2026. Estados: âœ… funcionando Â· ðŸ”„ en curso Â· â¸ï¸ apagado a propÃ³sito Â· ðŸ“¦ recibido, sin montar Â· ðŸ”œ planificado/pendiente Â· ðŸ—‘ï¸ a retirar/retirado Â· âš ï¸ sin confirmar

---

## 1. Red

| Dato | Valor |
|---|---|
| Router | Movistar WiFi 7 â€” `192.168.1.1` |
| Red local | `192.168.1.0/24`, mÃ¡scara 255.255.255.0 |
| Rango DHCP del router | `192.168.1.33` â€“ `192.168.1.199` |
| DNS que reparte el router por DHCP | `80.58.61.250` y `80.58.61.254` (Movistar). **No modificado** (decisiÃ³n) |
| Â¿El router responde como DNS en 192.168.1.1? | **No** de forma fiable (causa de la incidencia 13) |
| Rango para IP fijas del laboratorio | `192.168.1.240` en adelante (fuera del DHCP) |
| Dominio de Active Directory | `lab.internal` |
| Dominio para nombres locales (Pi-hole) | `home.arpa` (reservado para redes domÃ©sticas) |

---

## 2. Tabla de direcciones IP

| IP | Nombre | QuÃ© es | DÃ³nde vive | Estado |
|---|---|---|---|---|
| 192.168.1.1 | â€” | Router Movistar | â€” | âœ… |
| 192.168.1.40 (DHCP) | ws-cli01 | Cliente Windows 11 Enterprise (evaluaciÃ³n) | Hyper-V, PC | ðŸ—‘ï¸ se reconstruye en Proxmox |
| 192.168.1.240 | â€” | Libre (antes ubuntu-srv-01, retirado el 04/11/2026) | â€” | Libre |
| 192.168.1.241 | ws-dc01 | Controlador de dominio `lab.internal` + DNS | Hyper-V, PC | ðŸ—‘ï¸ se reconstruye en Proxmox (misma IP prevista) |
| 192.168.1.242 | **pve01** | AnfitriÃ³n Proxmox VE | OptiPlex 7020 | âœ… |
| 192.168.1.243 | **kuma01** | Uptime Kuma (CT 100) | pve01 | âœ… |
| 192.168.1.244 | **pihole01** | Pi-hole (CT 101) | pve01 | âœ… |
| 192.168.1.245 | lab01 | Contenedor de prÃ¡cticas de Linux (CT 102) | pve01 | ðŸ”œ |
| 192.168.1.246 | ansible01 | Nodo de control de Ansible | pve01 | ðŸ”œ propuesta |
| 192.168.1.247 | rocky-srv-01 | Rocky Linux, miembro del dominio | pve01 | ðŸ”œ propuesta |
| 192.168.1.248 | **docker01** | VM Debian 13 con Docker (NPM, Homepage, Paperless, n8n, GLPI) | pve01 | âœ… |

IP libres para servicios nuevos: `.240` y de `.249` en adelante. **Actualizar esta tabla cada vez.**

---

## 3. Tailscale

| Dato | Valor |
|---|---|
| Tailnet | MagicDNS activo (nombre en el gestor de contraseÃ±as; no se publica) |
| Router de subred | pve01 anuncia `192.168.1.0/24` (ruta aprobada). ReenvÃ­o IP en `/etc/sysctl.d/99-tailscale.conf` |
| DNS global de la tailnet | `192.168.1.244` (Pi-hole) con **Override DNS servers** activado |
| DNS en pve01 | `accept-dns=false`: pve01 **no** usa el DNS de la tailnet (usa 1.1.1.1). Ver incidencia 23 |
| Efecto conocido | El router de subred hace SNAT: en Pi-hole todas las consultas de la tailnet aparecen como `192.168.1.242` |

| Dispositivo | IP de Tailscale | Caducidad de clave | Notas |
|---|---|---|---|
| pve01 | 100.x.y.z | Desactivada | Servidor |
| pixel-10a | 100.x.y.z | Desactivada | MÃ³vil, uso remoto principal (chat del agente, Homepage) |
| ubuntusrv | 100.x.y.z | Desactivada | ðŸ—‘ï¸ âš ï¸ Confirmar que se quitÃ³ de la tailnet; si ya no estÃ¡, borrar esta fila |

---

## 4. Hardware

### Servidor: Dell OptiPlex 7020 SFF (pve01)

> ðŸ“¦ **Recibido y pendiente de montar** (sesiÃ³n de hardware en Yepes): RAM 2Ã—8 GB, HDD de 2 TB y el Kingston A400. En la misma sesiÃ³n: pasta tÃ©rmica, Memtest86+ (tambiÃ©n para descartar la RAM como causa de las caÃ­das) y SMART largo del HDD.

| Componente | Actual | Tras la sesiÃ³n de hardware ðŸ”œ |
|---|---|---|
| Procesador | Intel Core i5-4590 (4 nÃºcleos / 4 hilos, Haswell), grÃ¡fica HD 4600 (Quick Sync H.264) | â€” |
| Memoria | 16 GB DDR3-1600: 4Ã—4 GB (3 SK Hynix + 1 "Noname"). Uso tÃ­pico del anfitriÃ³n: ~5 GB | **24 GB**: 2Ã—8 GB Patriot Viper + 2Ã—4 GB Hynix (quitar el Noname). Un mÃ³dulo de 8 en cada canal |
| Disco 1 | **Toshiba TR200 240 GB** (SATA). SMART PASSED, sin errores, 26 588 h, desgaste 63, ~60 TB escritos. LVM: `root` 64 GB (`local`), `swap` 8 GB, `data` ~140 GB (`local-lvm`) | Proxmox, ISO, plantillas y contenedores ligeros. Sustituir a medio plazo |
| Disco 2 | â€” | **Kingston A400 240 GB**: salud 79 %, 3 465 h, 0 sectores malos, 27,9 TB escritos. â†’ **LVM-Thin `vmdata`** para las VM |
| Disco 3 | â€” | **HDD Samsung HD203WI 2 TB** (5400 rpm, fabricado en 2010). â†’ copias de seguridad + biblioteca de Jellyfin. Hacer SMART largo al montarlo |
| Lector DVD | Instalado | Se retira; su bahÃ­a aloja el Kingston con adaptador *caddy* |
| Red | 1 GbE integrada (Intel) | â€” |
| Fuente | Propietaria de Dell (~255 W) | â€” |
| Temperatura | En reposo 26â€“32 Â°C (Glances, `Package id 0`). ~69 Â°C en Memtest86+ | Cambiar pasta tÃ©rmica |

**BIOS:** VT-x y VT-d activados Â· UEFI Â· Secure Boot desactivado Â· SATA en AHCI Â· AC Recovery = Power On Â· errores de teclado desactivados.

### PC de casa (Windows 11 Pro)

| Componente | Valor |
|---|---|
| Placa | Gigabyte B550M K (BIOS F7) |
| Procesador | AMD Ryzen 7 5700X |
| Memoria | 16 GB |
| GrÃ¡fica | NVIDIA RTX 5060 (8 GB) â€” para Ollama/IA |
| C: (Windows) | **Samsung PM981 256 GB NVMe** (MZVLB256HBHQ-000L7), clonado el 02/11/2026. ~15 GB sin asignar al final |
| D: Juegos | SSD de 480 GB |
| Kingston A400 240 GB | Conectado pero **desactivado** por colisiÃ³n de firmas. No poner en lÃ­nea; va al servidor |
| AMD RX 6500 XT | Sin usar. No sirve para el servidor (no cabe, sin alimentaciÃ³n, sin codificador de vÃ­deo). Pensado venderla |
| Cuentas de Windows | `Martin` (PowerShell normal, **sin clave SSH** ðŸ”œ) Â· `Martin_` (administrador, tiene la clave Ed25519 autorizada en docker01) |
| Terminal | **Warp**: sesiones SSH a pve01 (root) y docker01 (martin). Modo Terminal; sin sugerencias de IA en las clases |

### PortÃ¡til de Aranjuez

| Dato | Valor |
|---|---|
| Clave SSH | Ed25519 propia (`portatil-aranjuez`), autorizada en docker01 |

---

## 5. Proxmox VE (pve01)

| Dato | Valor |
|---|---|
| VersiÃ³n | Proxmox VE 9.2.x, nÃºcleo 7.0.14-pve (Debian 13 "trixie") |
| InstalaciÃ³n | ext4 en el TR200 (`/dev/sda`). Almacenamientos: `local` (en `root`, 64 GB) y `local-lvm` (`data`, ~140 GB) |
| Repositorios | Enterprise desactivados Â· **pve-no-subscription** activado |
| DNS | Principal `1.1.1.1`, secundario `192.168.1.1`, dominio de bÃºsqueda `lab.internal`. **Nunca** un DNS alojado en el propio pve01 |
| Usuarios | `martin@pve` (usuario **de Proxmox**, Administrator en `/`, uso diario) Â· `root@pam` (emergencias). Ambos con **2FA TOTP** (Aegis) y cÃ³digos de recuperaciÃ³n |
| Tokens de API | **`martin@pve!homepage`** (*Privilege Separation*, `PVEAuditor` en `/`, solo lectura, lo usa Homepage) Â· **`martin@pve!n8n`** (*Privilege Separation*, `PVEAuditor` en `/`, solo lectura, lo usan el informe semanal y el agente) |
| SSH | Solo existe el usuario Linux `root`. Acepta **contraseÃ±a de root** (ðŸ”œ endurecer: solo clave, con la consola web como vÃ­a de emergencia; hacerlo estando en casa). La clave del PC no estÃ¡ autorizada para root. Termius en el mÃ³vil |
| Consola web | `~/.inputrc` con `set enable-bracketed-paste off`. noVNC no permite pegar: usar Tab para autocompletar |
| Notificaciones | Destino **`ntfy`** (webhook POST, URL `https://ntfy.sh/{{ secrets.topic }}`, cabecera `Title: {{ title }}`, cuerpo `{{ message }}`) â€” **Ãºnico aviso real**. Los avisos de mÃ¡s de 4 KB llegan como adjunto `attachment.txt`. `mail-to-root` **no entrega**: Gmail rechaza con `550 5.7.1` el correo directo desde IP domÃ©stica (ðŸ”œ relay SMTP o quitarlo). `default-matcher` â†’ ambos |
| Vigilancia externa | Healthchecks.io, chequeo `pve01`, periodo 5 min, gracia 5 min, integraciÃ³n ntfy. Cron en `/etc/cron.d/healthchecks` |
| Hardware | **Glances 4.3.1** (apt) como `glances-web.service` (`Restart=always`, `RestartSec=10`). Orden: `glances -w --disable-webui -B 192.168.1.242` â†’ solo API en `192.168.1.242:61208` (`/api/4/...`). Sensor de CPU: `Package id 0` (aviso 80 Â°C, crÃ­tico 100 Â°C) âœ… |
| Estabilidad | Varias caÃ­das sin apagado ordenado (30/10, 01/11 y tres el 04/11), probablemente cortes de luz. AC Recovery = Power On las recupera. Tras un reinicio, comprobar que **todo** volviÃ³ |
| Copias de seguridad | Tarea diaria **03:30**, modo snapshot, ZSTD, destino `local`. Incluye **100, 101 y 103** desde el 08/11/2026 (incidencia 24). âš ï¸ SelecciÃ³n todavÃ­a por mÃ¡quinas (`--all 0`): ðŸ”œ cambiar a **Modo de selecciÃ³n: Todo** (`--all 1`). RetenciÃ³n **temporal 3 diarias + 1 semanal** (copia completa â‰ˆ 6 GB: kuma01 0,6 Â· pihole01 0,3 Â· docker01 5,1); volver a 7+2 cuando estÃ©n en el HDD. RestauraciÃ³n probada, incluida una VM con base de datos |
| Copia fuera de casa | ðŸ”œ rclone con `crypt` hacia Google Drive (5 TB), cifrado en pve01; contraseÃ±a de cifrado tambiÃ©n en el gestor |
| Salud de discos | smartd activo; avisos vÃ­a notificaciones |

### Contenedores y mÃ¡quinas virtuales

| ID | Nombre | Tipo | SO | Recursos | IP | Estado |
|---|---|---|---|---|---|---|
| 100 | kuma01 | LXC sin privilegios (nesting=1), `onboot=1` | Debian 13 | 1 nÃºcleo Â· 512 MB Â· swap 512 Â· 4 GB | .243 | âœ… |
| 101 | pihole01 | LXC sin privilegios, `onboot=1` | Debian 13 | 1 nÃºcleo Â· 512 MB Â· swap 512 Â· 4 GB | .244 | âœ… |
| 102 | lab01 | LXC sin privilegios | Debian 13 | 1 nÃºcleo Â· 512 MB Â· 4 GB | .245 | ðŸ”œ (instantÃ¡nea `limpio`) |
| 103 | docker01 | VM, `onboot=1` | Debian 13 | 2 vCPU (`host`) Â· 3 GB con ballooning (mÃ­n. 1,5 GB) Â· 20 GB | .248 | âœ… Â· memoria al ~89 % (ðŸ”œ subir a 4 GB) |
| â€” | ansible01 | LXC | Debian 13 | ligero | .246 | ðŸ”œ |
| â€” | rocky-srv-01 | VM | Rocky Linux | ~1,5 GB | .247 | ðŸ”œ |
| â€” | ws-dc01 | VM | Windows Server (VirtIO) | 1,5â€“3 GB | .241 | ðŸ”œ reconstrucciÃ³n |
| â€” | ws-cli01 | VM | Windows 11 (vTPM, saltar requisitos CPU) | 4 GB | DHCP | ðŸ”œ reconstrucciÃ³n |

Plantilla descargada: `debian-13-standard_13.1-2_amd64.tar.zst`. Todos los contenedores usan DNS `1.1.1.1` y dominio `lab.internal`.

### Detalle de docker01 (VM 103)

| Dato | Valor |
|---|---|
| Hardware virtual | 2 vCPU tipo `host`, 3 GB con ballooning (mÃ­n. 1,5 GB), disco 20 GB VirtIO SCSI en `local-lvm` (*discard* + *SSD emulation*), red VirtIO, QEMU Guest Agent |
| Sistema | Debian 13 netinst (solo SSH y utilidades estÃ¡ndar) |
| Red | IP fija en `/etc/network/interfaces` (`ens18`), DNS `1.1.1.1` / `1.0.0.1` en `/etc/resolv.conf` (no usa Pi-hole) |
| SSH | Usuario `martin` con `sudo` y en el grupo `docker`. Solo clave Ed25519; `PermitRootLogin no` y `PasswordAuthentication no` en `sshd_config.d/01-hardening.conf`. Claves autorizadas: PC de casa (cuenta `Martin_`) y portÃ¡til de Aranjuez. âš ï¸ Confirmar `sudo sshd -T \| grep passwordauthentication` â†’ `no` |
| Docker | Docker Engine + Compose v5.6.1 desde el repositorio oficial de Docker |
| OrganizaciÃ³n | Un directorio por servicio: `/opt/<servicio>/compose.yaml`. Secretos en `.env` con permisos 600 |
| Puertos usados | 80 / 81 / 443 (NPM) Â· 3000 (Homepage) Â· 5678 (n8n) Â· 8000 (Paperless) Â· 8080 (GLPI â¸ï¸) |
| InstantÃ¡neas | `base`, `npm-ok`, `glpi-ok`, `paperless-ok`, `n8n-ok` âš ï¸ (si se hizo), `n8n-informe-ok` âš ï¸ (opcional) |

---

## 6. Servicios

### Uptime Kuma â€” kuma01

| Dato | Valor |
|---|---|
| URL | `http://192.168.1.243:3001` Â· `http://kuma.home.arpa` |
| VersiÃ³n | 2.5.8, instalada en `/opt/uptime-kuma` con Node.js de Debian |
| Servicio | `uptime-kuma.service` (systemd, `Restart=on-failure`) |
| Base de datos | SQLite (`/opt/uptime-kuma/data/kuma.db`) |
| Usuario | `administrador` (contraseÃ±a en el gestor) |
| Avisos | ntfy, prioridad alta, aplicado a todos los monitores |
| PÃ¡gina de estado | **`homelab`** (pÃºblica en la red local). La usan Homepage y n8n: `/api/status-page/homelab` (monitores) y `/api/status-page/heartbeat/homelab` (latidos y disponibilidad 24 h) |
| Restablecer contraseÃ±a | `cd /opt/uptime-kuma && npm run reset-password` |

Monitores (ID Â· nombre): 1 `pve01 - Web` Â· 2 `pve01-ping` Â· 3 `Internet Casa` Â· 6 `pihole01 - DNS` Â· 7 `docker01 - Ping` Â· 8 `npm - Panel` Â· 10 `paperless OK` Â· 11 `Homepage Ping` Â· 12 `GLANCES RENDIMIENTO` Â· 13 `N8N HTTP` Â· 5 `glpi - Web` â¸ï¸ en pausa (fuera de la pÃ¡gina de estado).
âš ï¸ Confirmar si existen todavÃ­a `ws-dc01 - DNS` y `ubuntu-srv-01` (no aparecen en la pÃ¡gina de estado). Si existen, pausar el primero y borrar el segundo.
ðŸ”œ Uptime Kuma vive en pve01: falta una vigilancia externa que avise si cae Ã©l mismo.

### Pi-hole â€” pihole01

| Dato | Valor |
|---|---|
| URL | `http://192.168.1.244/admin` Â· `http://pihole.home.arpa/admin` |
| VersiÃ³n | Pi-hole v6 |
| Uso | Solo DNS de la tailnet de Tailscale (no de toda la casa) |
| DNS ascendente | Cloudflare (1.1.1.1) |
| Listas | StevenBlack (por defecto) + **Hagezi Pro** (formato adblock, jsDelivr). ~303 000 dominios |
| Modo de privacidad | 0 (mostrar todo) |
| NTP de FTL | âœ… desactivado (`ntp.sync.active false`). Un LXC sin privilegios no puede cambiar la hora (incidencia 18) |
| Registros locales | âœ… `pve.home.arpa` â†’ .242 (directo, sin proxy) Â· `kuma`, `pihole`, `npm`, `glpi`, `paperless`, `homepage`, `n8n` `.home.arpa` â†’ .248 (NPM) |
| *App password* | Una propia para el widget de Homepage |
| ReenvÃ­o condicional | ðŸ”œ `lab.internal` â†’ DC (.241) cuando exista el nuevo DC |

### Nginx Proxy Manager â€” docker01

| Dato | Valor |
|---|---|
| URL | Panel `http://192.168.1.248:81` Â· `http://npm.home.arpa` |
| VersiÃ³n | 2.16 |
| UbicaciÃ³n | `/opt/npm/compose.yaml` Â· configuraciÃ³n y certificados en volÃºmenes |
| Proxy hosts | `kuma.home.arpa` â†’ .243:3001 Â· `pihole.home.arpa` â†’ .244:80 Â· `paperless.home.arpa` â†’ .248:8000 Â· `glpi.home.arpa` â†’ .248:8080 Â· `homepage.home.arpa` â†’ .248:3000 Â· `n8n.home.arpa` â†’ .248:5678 Â· `npm.home.arpa` â†’ 127.0.0.1:81 |
| Opciones | *Websockets* en Uptime Kuma, Paperless y n8n Â· *Block Common Exploits* activado Â· sin certificados (ðŸ”œ HTTPS con dominio propio) |
| ExcepciÃ³n | Proxmox **no** pasa por el proxy (ya va por HTTPS :8006) |

### Homepage â€” docker01

| Dato | Valor |
|---|---|
| URL | `http://homepage.home.arpa` Â· `http://192.168.1.248:3000` |
| VersiÃ³n | 2.7.0 (`ghcr.io/gethomepage/homepage:latest`) |
| UbicaciÃ³n | `/opt/homepage/compose.yaml` Â· configuraciÃ³n en `./config` (dueÃ±o `martin`, PUID/PGID 1000) |
| `HOMEPAGE_ALLOWED_HOSTS` | `homepage.home.arpa,192.168.1.248:3000` |
| Secretos | `/opt/homepage/.env` (permisos 600) con `HOMEPAGE_VAR_*` |
| DiseÃ±o (`settings.yaml`) | `title: Homelab` (ðŸ”œ cambiar a `BastiÃ³n`) Â· grupos Infraestructura y Servicios en filas de 3 Â· **Hardware** en fila de 4 |
| Widgets | Proxmox (token solo lectura) Â· Pi-hole v6 (*app password*) Â· Uptime Kuma (pÃ¡gina de estado `homelab`) Â· Paperless (token) Â· recursos de docker01 Â· grupo **Hardware** de pve01 vÃ­a Glances (`sensor:Package id 0`, `cpu`, `memory`, `fs:/`) âœ… |
| Enlaces | `href` con nombres `*.home.arpa`; `siteMonitor` con IP (docker01 no usa Pi-hole) |
| Estilo | GuÃ­a propia (oscuro + rojo `#E02B2F`): `color: red` + `config/custom.css` |
| Seguridad | **Sin socket de Docker montado** |

### n8n â€” docker01

| Dato | Valor |
|---|---|
| URL | `http://n8n.home.arpa` Â· `http://192.168.1.248:5678` |
| VersiÃ³n | 2.46.6 (`docker.n8n.io/n8nio/n8n`) |
| UbicaciÃ³n | `/opt/n8n/compose.yaml` Â· volumen `n8n_data` Â· `.env` (600) con `N8N_ENCRYPTION_KEY` |
| Variables | En `compose.yaml`: `N8N_HOST=n8n.home.arpa`, `N8N_WEBHOOK_URL=http://n8n.home.arpa/`, `N8N_SECURE_COOKIE=false`, `GENERIC_TIMEZONE` y `TZ` = `Europe/Madrid` |
| Credenciales (cifradas en n8n) | `Google Gemini(PaLM) Api` (clave del proyecto de AI Studio) Â· `Proxmox API (n8n)` (Header Auth, *Allowed domains*: 192.168.1.242) Â· Basic Auth del chat del agente |
| Vigilancia | Monitor `N8N HTTP` en Kuma (`/healthz`) |

**Flujos:**

| Flujo | QuÃ© hace | Estado |
|---|---|---|
| Informe semanal BastiÃ³n | Lunes 9:00 Â· Kuma (monitores + disponibilidad 24 h) + Glances (memoria, sensores, disco) + Proxmox (tareas `vzdump` de 7 dÃ­as) â†’ nodo Code `Preparar datos` â†’ **Gemini 3.6 Flash** â†’ ntfy (`Title: Informe semanal homelab`, ðŸ”œ cambiar a BastiÃ³n) | âœ… publicado; primera ejecuciÃ³n automÃ¡tica el 09/11 9:00 |
| Herramienta - Estado BastiÃ³n | Subflujo (*When Executed by Another Workflow*) con la misma cadena hasta `Preparar datos`; devuelve el texto `informe` | âœ… publicado |
| Agente BastiÃ³n | Chat pÃºblico (*Hosted Chat*, **Basic Auth**) â†’ AI Agent con **Gemini 3.8 Flash** + Simple Memory (10) Â· herramientas de solo lectura: `estado_general` (subflujo) y `maquinas_proxmox` (`/cluster/resources?type=vm`) | âœ… publicado Â· URL `http://n8n.home.arpa/webhook/<id>/chat` Â· acceso directo en el Pixel |

**API de Gemini (Google AI Studio):** proyecto "Proyecto de Gemini", nivel 1 Â· ~13 â‚¬ de crÃ©dito Â· gasto real a 09/11: 0,09 â‚¬ cubiertos por el descuento (â‰ˆ0,01 â‚¬ por llamada) Â· ðŸ”œ lÃ­mite de inversiÃ³n mensual de **3 â‚¬** âš ï¸ confirmar que se puso Â· "Logs and Datasets" **activado** (Google guarda peticiones y respuestas): **nada personal ni secretos a Gemini**.

### Paperless-ngx â€” docker01

| Dato | Valor |
|---|---|
| URL | `http://paperless.home.arpa` Â· `http://192.168.1.248:8000` |
| VersiÃ³n | 3.4.1 + PostgreSQL 18 + Valkey (Compose oficial, variante PostgreSQL) |
| UbicaciÃ³n | `/opt/paperless/` Â· volÃºmenes con nombre gestionados por Docker |
| ConfiguraciÃ³n | OCR espaÃ±ol + inglÃ©s Â· `Europe/Madrid` Â· `PAPERLESS_ALLOWED_HOSTS` y `PAPERLESS_CSRF_TRUSTED_ORIGINS` con nombre e IP:puerto Â· clave secreta en `docker-compose.env` con permisos 600 |
| Token de API | Uno para el widget de Homepage |
| Estado | ðŸ”„ **En evaluaciÃ³n frente a Google Drive**. Se deja encendido. Posible integraciÃ³n: rclone sincroniza una carpeta de Drive con la carpeta *consume* de Paperless. Si no se usa, se apaga como GLPI |

### GLPI â€” docker01 â¸ï¸

| Dato | Valor |
|---|---|
| URL | `http://glpi.home.arpa` Â· `http://192.168.1.248:8080` |
| VersiÃ³n | Imagen oficial `glpi/glpi` + MariaDB 11.4 (versiÃ³n fijada) |
| UbicaciÃ³n | `/opt/glpi/compose.yaml` Â· credenciales en `.env` (permisos 600) Â· `storage/glpi` con dueÃ±o 33:33 |
| Estado | â¸ï¸ **Apagado desde 04/11/2026** (`docker compose stop`), datos conservados e instantÃ¡nea `glpi-ok`. Monitor en pausa y fuera de Homepage |
| Encender | `cd /opt/glpi && docker compose start` |

---

## 7. Accesos rÃ¡pidos (marcadores)

| Servicio | Enlace |
|---|---|
| Homepage (entrada principal) | `http://homepage.home.arpa` Â· `http://192.168.1.248:3000` |
| Agente BastiÃ³n (chat) | `http://n8n.home.arpa/webhook/<id>/chat` (Basic Auth) |
| Proxmox | `https://192.168.1.242:8006` Â· `https://pve.home.arpa:8006` |
| Proxmox (respaldo por IP de Tailscale) | `https://100.x.y.z:8006` |
| Uptime Kuma | `http://kuma.home.arpa` Â· `http://192.168.1.243:3001` |
| Pi-hole | `http://pihole.home.arpa/admin` Â· `http://192.168.1.244/admin` |
| Nginx Proxy Manager | `http://npm.home.arpa` Â· `http://192.168.1.248:81` |
| n8n (editor) | `http://n8n.home.arpa` Â· `http://192.168.1.248:5678` |
| Paperless-ngx | `http://paperless.home.arpa` Â· `http://192.168.1.248:8000` |
| GLPI â¸ï¸ | `http://glpi.home.arpa` Â· `http://192.168.1.248:8080` |
| Glances (API) | `http://192.168.1.242:61208/api/4/status` |
| Consola de Tailscale | `https://login.tailscale.com/admin` |
| Healthchecks | `https://healthchecks.io` |
| Google AI Studio (gasto de Gemini) | `https://aistudio.google.com` |

Fuera de casa funcionan con Tailscale activo en el mÃ³vil (router de subred). Los nombres `*.home.arpa` solo funcionan con Tailscale (Pi-hole); en casa sin Tailscale, usar la IP.

---

## 8. DÃ³nde estÃ¡n los secretos (solo la ubicaciÃ³n)

| Secreto | DÃ³nde |
|---|---|
| ContraseÃ±as de `root@pam`, `martin@pve`, contenedores, Kuma, Pi-hole, NPM, Paperless, GLPI, n8n | Gestor de contraseÃ±as |
| CÃ³digos de recuperaciÃ³n 2FA de Proxmox | Gestor de contraseÃ±as / copia en papel |
| Nombre del tema de ntfy | Gestor de contraseÃ±as (en Proxmox estÃ¡ como *secret*; en n8n dentro de la URL del nodo ntfy). ðŸ”œ Cambiarlo: apareciÃ³ en capturas |
| URL de ping de Healthchecks | Panel de Healthchecks y `/etc/cron.d/healthchecks` |
| Token de Proxmox para Homepage, *app password* de Pi-hole y token de Paperless | Gestor de contraseÃ±as y `/opt/homepage/.env` (permisos 600) |
| Token de Proxmox para n8n (`martin@pve!n8n`) | Gestor de contraseÃ±as y credencial cifrada en n8n |
| Clave de la API de Gemini | Gestor de contraseÃ±as y credencial cifrada en n8n |
| Usuario y contraseÃ±a del chat del agente (Basic Auth) | Gestor de contraseÃ±as y credencial cifrada en n8n |
| `N8N_ENCRYPTION_KEY` | `/opt/n8n/.env` (600) y gestor de contraseÃ±as. Sin ella no se pueden descifrar las credenciales de n8n al restaurar |
| Credenciales de bases de datos (GLPI, Paperless) | `.env` / archivo de configuraciÃ³n de cada servicio en `/opt/<servicio>/` (permisos 600) |
| Clave privada SSH | Solo en cada cliente (PC `Martin_`, portÃ¡til de Aranjuez y mÃ³vil). Nunca se copia entre equipos |
| ContraseÃ±a de cifrado de rclone ðŸ”œ | Gestor de contraseÃ±as (ademÃ¡s de en pve01). Sin ella, las copias de Drive son irrecuperables |

