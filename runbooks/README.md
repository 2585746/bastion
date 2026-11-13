# Runbooks de Bastión

Guías paso a paso, una por fase del [plan maestro](../docs/plan-maestro.md). Cada una indica dónde se ejecuta, explica cada comando y termina con una lista de verificación. El avance se marca en el [ROADMAP](../docs/ROADMAP.md).

## Orden y dependencias

| # | Runbook | Fase | Necesita antes | Dónde se hace | Tiempo |
|---|---|---|---|---|---|
| 01 | [Copia cifrada a Google Drive](01-copia-externa-rclone.md) | A1 | — | Remoto (Tailscale) | 45 min |
| 02 | [Cerrar la infraestructura](02-cierre-infraestructura.md) | A2–A7 | 01 | Remoto, salvo el SSH de pve01 (en casa) | 2 h |
| 03 | [Sesión de hardware](03-hardware.md) | B | 01 (copia fresca) | En casa, con el servidor delante | Una tarde |
| 04 | [Linux y Ansible](04-linux-y-ansible.md) | C | — | Remoto | Continuo |
| 05 | [Web pública en Google Cloud](05-web-publica-google-cloud.md) | D | Sitio web y dominio | Remoto | 3 h |
| 06 | [SOC con Wazuh](06-soc-wazuh.md) | E | 03 (24 GB) | Remoto + PC de casa | Varias sesiones |
| 07 | [Active Directory](07-active-directory.md) | F | 03 | Remoto | 2 sesiones |
| 08 | [Azure y Terraform](08-azure-y-terraform.md) | G | 04 (rol `bootstrap`), 05 | Remoto | 2 sesiones |
| 09 | [Migración y cierre de Google](09-migracion-y-cierre-google.md) | H | 05, 08 | Remoto | 3 h |

El sitio web se construye con el prompt de [`prompts/claude-code-portafolio.md`](../prompts/claude-code-portafolio.md) y puede hacerse en cualquier momento: no depende de ninguna fase.

```
01 ─► 02 ─► 03 ─┬─► 06 (SOC) ◄── honeypot en la nube (05)
                └─► 07 (Active Directory)
04 (Ansible: rol bootstrap) ─► 05 (web) ─► 08 (Azure) ─► 09 (Oracle y cierre)
prompt del portafolio ─► 05
```

## Cómo usar un runbook

1. **Léelo entero antes de empezar.** Sabrás cuánto dura y si necesitas estar en casa.
2. **Un comando, una comprobación.** No sigas si la salida no es la esperada (regla 1: leer la salida literal).
3. **Instantánea antes de cambios grandes** en una VM: `qm snapshot <id> antes-de-<cambio>`.
4. **Al terminar:** casillas de verificación, inventario, ROADMAP y *commit* en GitHub.
5. **Si algo falla:** documenta la incidencia con el formato de siempre (síntoma → diagnóstico → causa → solución → aprendizaje). La siguiente es la **26**.

## Avisos generales

- **Versiones y precios cambian.** Los runbooks de nube y de Wazuh indican dónde comprobar la versión o el coste actual antes de ejecutar.
- **Sustituye los marcadores** (`sdX`, `tudominio.com`, `vX.Y.Z`, `bastion-cloud-XXXX`, `TU-TEMA-NTFY`) por tus valores. Ninguno de los reales se sube a GitHub.
- **Nada de secretos en los comandos que pegues en un chat o en una captura.**
- **Lo que toca el acceso al servidor** (SSH de pve01, red, cortafuegos) se hace en casa, con la consola web como vía de emergencia.
