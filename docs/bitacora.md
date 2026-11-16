# Bastión — Bitácora

> Diario fechado de cambios en la infraestructura y en el plan. Lo más reciente, arriba.

---

### 2026-11-16 · Claude Code
- Hecho: añadidas las instrucciones comunes para agentes (`agentes-bastion-v2/`), limpiados los marcadores de IP de Tailscale del inventario y dos falsos positivos del escaneo de privacidad. Instalado el plugin de diseño Impeccable: `PRODUCT.md` aquí; `DESIGN.md` y crítica (20/28) en `bastion-web`. Publicada en la web la mejora de `/bastion` (commit `6ba8945`): resumen por capa con detalle plegado, «entrada de guardia» con cifras, cierre con enlaces, pestañas adaptadas a móvil.
- Decisiones: la web muestra las incidencias publicadas contadas desde `incidencias.json` (hoy 3), no la cifra escrita a mano (27). Versión de Proxmox, monitores y fecha de los datos viven en `facts` (`src/data/bastion.ts`).
- Pendiente: actualizar `facts.asOf` en la web cuando cambien esos datos; revisar los resúmenes nuevos de las capas; decidir si los archivos de agentes pasan a la raíz para que los agentes los carguen solos.

## 16/11/2026 — Fase B: RAM a 24 GB; el Kingston sale del plan

- **Hecho:** RAM de `pve01` ampliada a **24 GB**. Con esto hay memoria para Wazuh y quedan desbloqueadas las fases E (SOC) y F (Active Directory).
- **Cambio de plan:** el **Kingston A400 ya no va al servidor**. Se quita el almacenamiento `vmdata` del plan; los discos de las VM siguen en `local-lvm` (SSD del sistema).
- **Pendiente:** conectar el **HDD de 2 TB** para las copias (y volver la retención a 7 diarias + 2 semanales) y hacerle el SMART largo.
- **Por revisar:** los runbooks que todavía usan `vmdata` (`03-hardware`, `04-linux-y-ansible`, `06-soc-wazuh`, `07-active-directory`), el inventario y el ROADMAP.
