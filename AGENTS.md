# Bastión: instrucciones para agentes (Claude Code y Gemini CLI)

Este archivo es la única fuente de instrucciones para cualquier agente de IA que trabaje en este repositorio. `CLAUDE.md` lo importa y Gemini CLI lo lee mediante `.gemini/settings.json`. No dupliques estas reglas en otros archivos: si hay que cambiar una regla, se cambia aquí.

## Quién y qué

- Dueño: Martín, técnico de soporte IT y estudiante de ASIR. Háblale en español neutro (no de España).
- Bastión es su homelab en casa: servidor Proxmox `pve01` (Dell OptiPlex 7020) con `kuma01` (CT100), `pihole01` (CT101), `lab01` (CT102) y `docker01` (VM103: Nginx Proxy Manager, Homepage, n8n y GLPI). Paperless está apagado.
- Repositorios: `bastion` (este, documentación) y `bastion-web` (web martindeprada.dev, Astro en GitHub Pages).

## Dónde está la verdad

Lee al empezar, en este orden:

1. `docs/00-estado-actual.md`: cómo está todo hoy.
2. `docs/plan-maestro.md`: fases, decisiones y pendientes.
3. `docs/bitacora.md`: lo que se hizo en las últimas sesiones y con qué agente.

Para detalles: `docs/01-inventario.md`, `docs/03-incidencias.md`, `docs/runbooks/` y `docs/web-pendientes.md`.

Si lo que dice Martín contradice estos documentos, pregúntale cuál vale y corrige el documento.

## Ritual de cada sesión

Al empezar:

1. `git pull`: otro agente pudo trabajar desde la última vez.
2. Lee los tres documentos de arriba y resume en 3–5 líneas en qué punto estamos y qué sigue.

Al terminar, aunque la sesión haya sido corta:

1. Actualiza `docs/00-estado-actual.md` si cambió el estado de algo.
2. Añade una entrada **al principio** de `docs/bitacora.md` con el formato de abajo.
3. Si cambió una fase o una decisión, actualízala en `docs/plan-maestro.md`.
4. Pasa el escaneo de privacidad, haz commit y push.

Formato de la bitácora:

```
### AAAA-MM-DD · Claude Code | Gemini CLI
- Hecho: ...
- Decisiones: ...
- Pendiente: ...
```

## Reglas

- **Un solo agente a la vez en cada repositorio.** Si `git pull` trae cambios inesperados o hay conflicto, para y avisa a Martín.
- **Commits** con prefijo `[claude]` o `[gemini]`, en español, diciendo qué cambió y por qué.
- **Servidores:** no ejecutes comandos en `pve01` ni en las VM o CT por tu cuenta. Prepara los comandos, explica qué hace cada uno y espera a que Martín los ejecute y te pegue la salida (salvo que te dé permiso explícito en esa sesión). Antes de cualquier cambio en un servidor, propón un snapshot.
- **Secretos:** nunca escribas en el repositorio tokens, contraseñas, claves, IP de Tailscale (100.x.x.x), topics de ntfy ni IDs de proyectos de Google Cloud. Usa marcadores como `<TOKEN>`.
- **Escaneo antes de cada commit** (después de `git add`). Si sale cualquier línea, no hagas commit:

  ```
  git --no-pager grep --cached -nIE "ntfy\.sh/[A-Za-z0-9_-]+|PVEAPIToken[=]|BEGIN[ ]OPENSSH|password\s*[:=]|100\.[0-9]+\.[0-9]+\.[0-9]+|bastion-cloud-[0-9]+"
  ```

- **Incidencias:** se numeran en `docs/03-incidencias.md`. La 28 está reservada; la siguiente libre es la 30. Formato `### Incidencia N — título` con los apartados Síntoma, Diagnóstico, Causa, Solución y Aprendizaje. Cada incidencia nueva se crea también en GLPI, que es la fuente de la web (flujo de n8n).
- **Web (`bastion-web`): en pausa.** No la toques salvo que Martín lo pida expresamente. Los ajustes ya decididos están en `docs/web-pendientes.md`. Ninguna automatización debe editar `src/pages/bastion/incidencias/index.astro`: los datos van solo en `src/data/bastion/incidencias.json`.
- **Estilo visual** de cualquier cosa con diseño: tema oscuro, acento rojo `#E02B2F`, fuente Poppins, esquinas rectas.
