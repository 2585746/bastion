# Bastión

**Laboratorio de infraestructura y ciberseguridad autoalojado, con IA integrada y nube híbrida.**

Proyecto personal de Martín De Prada. Un servidor físico en casa con Proxmox VE, extendido a la nube (Google Cloud → Oracle/Azure) con infraestructura como código, un SOC con Wazuh y automatización asistida por IA — sin exponer ningún puerto de la red doméstica.

> Antes llamado "Homelab-ASIR-Practica". En migración al nombre **Bastión**.

---

## Qué hay montado

- **Hipervisor:** Proxmox VE 9.2 en un Dell OptiPlex 7020, con 2FA, copias diarias probadas y vigilancia externa.
- **Red:** acceso remoto por Tailscale (router de subred), DNS propio con Pi-hole, proxy inverso con Nginx Proxy Manager. **Cero puertos abiertos en el router.**
- **Observabilidad:** Uptime Kuma, Glances y Healthchecks.io, con alertas al móvil por ntfy.
- **Automatización + IA:** n8n con la API de Gemini genera un informe semanal del estado y expone un agente de operaciones por chat (solo lectura) consultable desde el móvil.
- **Documentación:** inventario como fuente de verdad, 25 incidencias con su diagnóstico y memorias por fase.

## Hacia dónde va

Copia cifrada fuera de casa (3-2-1) → web pública con WAF en la nube → **SOC con Wazuh** que detecta ataques controlados y reales (honeypot) y los resume con IA → Active Directory / Entra ID → migración entre nubes con Terraform y Ansible.

El plan completo está en [`docs/plan-maestro.md`](docs/plan-maestro.md) y el avance en [`docs/ROADMAP.md`](docs/ROADMAP.md).

## Tecnologías

Linux · Windows Server · Active Directory · Proxmox · Docker · Ansible · Terraform · Google Cloud · Azure · Tailscale/WireGuard · Pi-hole · Nginx · Wazuh · n8n · API de Gemini · Bash

## Estructura

| Carpeta | Contenido |
|---|---|
| `docs/` | Estado actual, inventario, incidencias, plan maestro, memorias |
| `runbooks/` | Guías paso a paso, una por fase (empieza por `runbooks/README.md`) |
| `prompts/` | Prompt para construir el portafolio web con Claude Code |
| `compose/` | Un `compose.yaml` por servicio (sin secretos) |
| `ansible/` | Playbooks y roles |
| `terraform/` | Infraestructura de nube |

## Nota de seguridad

Este repositorio no contiene contraseñas, claves, tokens, el nombre de la red Tailscale ni los temas de notificación. Las IP que aparecen son privadas (RFC 1918) y no accesibles desde internet. La parte ofensiva del laboratorio (SOC) se ejecuta contra infraestructura propia en redes aisladas.

## Licencia

MIT (por definir).
