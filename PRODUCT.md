# Product

<!-- impeccable:product-schema 1 -->

## Platform

web

## Stack

La web es `bastion-web` (repositorio aparte, en pausa): Astro con salida estática en GitHub Pages, publicada en martindeprada.dev. Este repositorio (`bastion`) solo contiene la documentación y este archivo es una referencia hasta que se retome la web.

## Users

- Reclutadores y técnicos de IT que evalúan el portafolio de Martín (estudiante de ASIR y técnico de soporte IT): quieren ver qué ha montado y cómo resuelve incidencias.
- Martín y los agentes de IA (Claude Code, Gemini CLI) que trabajan con él: retoman el trabajo y repiten procedimientos desde el estado actual, el plan, la bitácora y los runbooks.

## Product Purpose

Bastión es el homelab de Martín en casa: un servidor Proxmox (`pve01`) con contenedores y una VM de Docker (Nginx Proxy Manager, Homepage, n8n y GLPI). La web muestra ese proyecto como evidencia de su trabajo: qué hay montado, las incidencias resueltas y los runbooks.

## Operating Context

- Las incidencias se numeran en `docs/03-incidencias.md` y se registran también en GLPI, que es la fuente de datos de la web (flujo de n8n).
- Los datos de incidencias van solo en `src/data/bastion/incidencias.json` de `bastion-web`; ninguna automatización edita `src/pages/bastion/incidencias/index.astro`.
- Los ajustes ya decididos de la web están en `docs/web-pendientes.md`.

## Capabilities and Constraints

- **Privacidad estricta:** nunca se publican IP (incluidas las de Tailscale), tokens, contraseñas, claves, topics de ntfy ni IDs de proyectos de Google Cloud. Se usan marcadores como `<TOKEN>`.
- **Idiomas:** español neutro (no de España) e inglés. La versión en inglés se redacta, no se traduce palabra por palabra.
- Sin decidir (no confirmado en esta entrevista): analítica, cookies y recursos de terceros.

## Brand Commitments

El acento rojo `#E02B2F` es la seña de identidad y se mantiene. Desde el 16/11/2026 la web usa tema claro inspirado en impeccable.style (decisión de Martín); el sistema completo está en `DESIGN.md` de `bastion-web` y la regla resumida en `AGENTS.md`.

## Evidence on Hand

- Incidencias reales documentadas en `docs/03-incidencias.md`.
- Inventario en `docs/01-inventario.md`, runbooks en `runbooks/` y estado en `docs/00-estado-actual.md`.
- No hay testimonios, clientes ni métricas que citar; no se inventan.

## Product Principles

1. Cada afirmación se apoya en algo verificable: se distingue lo hecho de lo planificado.
2. La privacidad va antes que el detalle técnico: un dato sensible publicado desacredita un proyecto de seguridad.
3. El contenido sirve a dos lectores a la vez: quien evalúa rápido y quien necesita reproducir el procedimiento.
