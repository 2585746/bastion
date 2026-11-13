# Bastión — ROADMAP y seguimiento

> Tablero de progreso. Cada casilla marcada es un hito verificable (un *commit*, una incidencia cerrada, un servicio en marcha).
> Actualizado: 10/11/2026.

## Leyenda
`[x]` hecho · `[~]` en curso · `[ ]` pendiente

---

## Hecho (base de la infraestructura)

- [x] Servidor OptiPlex 7020 validado (Memtest86+, SMART) y Proxmox VE 9.2 en UEFI
- [x] Tailscale como router de subred, sin puertos abiertos en casa
- [x] Pi-hole (DNS + nombres `*.home.arpa`) para la tailnet
- [x] Uptime Kuma (10 monitores) + ntfy + Healthchecks.io externo
- [x] Glances en el anfitrión para el hardware
- [x] Copias diarias de las 3 máquinas, restauración probada
- [x] docker01 endurecida con Docker Compose
- [x] Nginx Proxy Manager, Homepage, Paperless, GLPI (apagado)
- [x] n8n: informe semanal con Gemini + agente de operaciones por chat
- [x] Inventario, 25 incidencias y memorias documentadas

---

## Fase A — Cerrar la infraestructura (esta semana)

- [~] A1 · Copia cifrada a Drive con rclone (3-2-1) — runbook listo, falta ejecutar
- [ ] A2 · Tarea de copias en "Modo de selección: Todo"
- [ ] A3 · docker01 a 4 GB de RAM
- [ ] A4 · Paperless: integrar con Drive o apagar
- [ ] A5 · SSH de pve01 solo con clave (en casa)
- [ ] A6 · Quitar `mail-to-root` y cambiar el tema de ntfy
- [ ] A7 · Subir Bastión a GitHub con la estructura nueva

## Fase B — Hardware (visita a Yepes)

- [ ] B1 · RAM a 24 GB
- [ ] B2 · Kingston A400 como `vmdata`, mover docker01
- [ ] B3 · HDD de 2 TB para copias, retención a 7+2
- [ ] B4 · Pasta térmica, Memtest86+, SMART largo del HDD

## Fase C — Linux + Ansible

- [ ] C1 · Clases de Linux en lab01 (12 clases)
- [ ] C2 · Ansible: inventario, playbook de actualizaciones, bootstrap
- [ ] C3 · Playbooks en GitHub

## Fase D — Nube 1: web pública (Google Cloud)

- [ ] D1 · Presupuesto y alerta en Google Cloud; apuntar fecha de caducidad
- [ ] D2 · Sitio Astro (portafolio + blog + incidencias) en GitHub Pages
- [ ] D3 · VM en Google Cloud (Madrid), en la tailnet
- [ ] D4 · Dominio del Student Pack + HTTPS por DNS
- [ ] D5 · WAF (OWASP) + CrowdSec en la web

## Fase E — SOC (tras la fase B)

- [ ] E1 · Wazuh en una VM de Bastión
- [ ] E2 · Red aislada + máquina objetivo con agente
- [ ] E3 · Kali en el PC, aislada, para pruebas controladas
- [ ] E4 · Honeypot Cowrie en la nube → Wazuh
- [ ] E5 · Informe diario de amenazas (Wazuh → n8n → Gemini → ntfy)

## Fase F — Active Directory en Proxmox

- [ ] F1 · DC `lab.internal` reconstruido en orden
- [ ] F2 · Cliente Windows nuevo
- [ ] F3 · rocky-srv-01 unido al dominio

## Fase G — Laboratorios bajo demanda (Azure/AWS)

- [ ] G1 · Azure for Students activado (2 semanas antes de que caduque Google)
- [ ] G2 · Laboratorio de Entra ID / AD en Azure
- [ ] G3 · Mismos Ansible/Terraform que en Google (prueba de migración)

## Fase H — Oracle y cierre de Google

- [ ] H1 · Web y monitorización a Oracle Always Free
- [ ] H2 · Copia de datos de la nube al homelab
- [ ] H3 · Borrar todos los recursos de Google Cloud

---

## Decisiones abiertas (tuyas)

- [ ] Nombre y dominio definitivos (propuesta: Bastión)
- [ ] Renombrar el repositorio `Homelab-ASIR-Practica`
- [ ] Datos de contacto que aparecen en la web pública
- [ ] Futuro de Paperless
