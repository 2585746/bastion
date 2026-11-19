# Bastión — Bitácora

> Diario fechado de cambios en la infraestructura y en el plan. Lo más reciente, arriba.

---

### 2026-11-19 · Claude Code (fase I)
- Hecho: documentada la fase I. Decisión: túnel WireGuard saliente desde casa (`edge01`, CT 105) hacia un VPS en Google Cloud (`bastion-edge`, Madrid, e2-micro) con Caddy; sin puertos abiertos en el router; Cloudflare descartado por ahora. Creado `runbooks/10-tunel-wireguard-vps.md` (`wg0.conf` de los dos lados, `nftables` de `edge01`, Caddyfile y servicio de prueba, sin claves ni IP públicas). Inventario con CT 105, la VM y la red `10.66.0.0/24`; ROADMAP con I1–I3 hechos y plan maestro con la decisión.
- Decisiones: «0 puertos abiertos» sigue siendo cierto; el riesgo se limita con `forward` denegado en `edge01` y un cortafuegos que solo deja entrar el puerto del servicio por el túnel.
- Pendiente: I4 (adaptar la web: alojamiento, despliegue y monitor en Kuma); añadir `edge01` y la URL de prueba a Uptime Kuma; comprobar la lista de verificación del runbook 10 con las máquinas reales.

---

### 2026-11-19 · Claude Code
- Hecho: el HDD Samsung de 2 TB murió el 18/11 (no se detecta en ningún puerto y hace ruido de clic). La fase B queda pendiente de comprar otro disco. ROADMAP actualizado (B1 hecho, B2 descartado, B3 pendiente de disco nuevo) y runbooks 03, 04, 06 y 07 pasados de `vmdata` a `local-lvm`. Los archivos de agentes (`AGENTS.md`, `CLAUDE.md`, `GEMINI.md`, `.gemini/`) pasan a la raíz y se borra `agentes-bastion-v2/`.
- Decisiones: se descarta el rediseño claro de la web (`bastion-web` vuelve a `main`, diseño oscuro, conservando las mejoras de Impeccable): la regla de estilo vuelve a tema oscuro, acento rojo `#E02B2F`, fuente Poppins y esquinas rectas (colores planeados originalmente), en `AGENTS.md` y `PRODUCT.md`.
- Pendiente: comprar el disco de 2 TB y retomar B3/B4; retomar la web cuando se decida (el rediseño claro quedó solo en la rama `rediseno-claro` de `bastion-web`, sin fusionar; `main` mantiene el diseño oscuro con las mejoras de Impeccable, commit `6ba8945`).

### 2026-11-16 · Claude Code
- Hecho: añadidas las instrucciones comunes para agentes (`agentes-bastion-v2/`), limpiados los marcadores de IP de Tailscale del inventario y dos falsos positivos del escaneo de privacidad. Instalado el plugin de diseño Impeccable: `PRODUCT.md` aquí; `DESIGN.md` y crítica (20/28) en `bastion-web`. Publicada en la web la mejora de `/bastion` (commit `6ba8945`): resumen por capa con detalle plegado, «entrada de guardia» con cifras, cierre con enlaces, pestañas adaptadas a móvil.
- Decisiones: la web muestra las incidencias publicadas contadas desde `incidencias.json` (hoy 3), no la cifra escrita a mano (27). Versión de Proxmox, monitores y fecha de los datos viven en `facts` (`src/data/bastion.ts`).
- Pendiente: actualizar `facts.asOf` en la web cuando cambien esos datos; revisar los resúmenes nuevos de las capas; decidir si los archivos de agentes pasan a la raíz para que los agentes los carguen solos.

## 16/11/2026 — Fase B: RAM a 24 GB; el Kingston sale del plan

- **Hecho:** RAM de `pve01` ampliada a **24 GB**. Con esto hay memoria para Wazuh y quedan desbloqueadas las fases E (SOC) y F (Active Directory).
- **Cambio de plan:** el **Kingston A400 ya no va al servidor**. Se quita el almacenamiento `vmdata` del plan; los discos de las VM siguen en `local-lvm` (SSD del sistema).
- **Pendiente:** conectar el **HDD de 2 TB** para las copias (y volver la retención a 7 diarias + 2 semanales) y hacerle el SMART largo.
- **Por revisar:** los runbooks que todavía usan `vmdata` (`03-hardware`, `04-linux-y-ansible`, `06-soc-wazuh`, `07-active-directory`), el inventario y el ROADMAP.
