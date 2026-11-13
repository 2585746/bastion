# Bastión — Registro de incidencias

> Formato: síntoma → diagnóstico → causa → solución → aprendizaje.
> Las incidencias 1–12 son de las fases 1, 1b y 2 (Hyper-V); el detalle completo está en la memoria Word de esas fases. De la 13 en adelante, fase 3 y fase A (Proxmox).
> **27 incidencias registradas. La siguiente nueva será la 28.**
> Última actualización: 13/11/2026.

---

## Resumen

| Nº | Fecha | Área | Incidencia | Causa raíz en una línea |
|---|---|---|---|---|
| 1 | Oct 2026 | Hyper-V | La VM no arranca | Virtualización (SVM) desactivada en la BIOS |
| 2 | Oct 2026 | SSH | "Permission denied" con la contraseña correcta | Sintaxis incorrecta del comando ssh |
| 3 | Oct 2026 | SSH | SSH "inactive (dead)" pero funciona | Activación por socket de systemd |
| 4 | Oct 2026 | Linux | Archivo bloqueado | nano huérfano tras cortarse la sesión SSH |
| 5 | Oct 2026 | Hyper-V | El portapapeles escribe caracteres incorrectos | La consola simula pulsaciones de teclado |
| 6 | Oct 2026 | Hyper-V | Pérdida de configuración | Punto de control automático revertido |
| 7 | Oct 2026 | SSH | El endurecimiento no surte efecto | En `sshd_config.d/` gana el primer archivo |
| 8 | Oct 2026 | Linux | Cron fuera de hora | Servidor en UTC |
| 9 | Oct 2026 | AD | El DC conserva su nombre automático | Promovido antes de renombrar |
| 10 | Oct 2026 | AD / red | El cliente no llega al DC | El DC perdió la IP fija |
| 11 | Oct 2026 | AD / DNS | Registros DNS hacia una IP muerta | El DC no retira sus registros antiguos |
| 12 | Oct 2026 | Hyper-V | VM congelada en el arranque | Fusión de un punto de control en segundo plano |
| 13 | 30/10/2026 | Red / DNS | pve01 no resuelve nombres | El router Movistar no responde como DNS |
| 14 | Oct 2026 | Proxmox | El pegado falla en la consola web | *Bracketed paste* de bash |
| 15 | Oct 2026 | Windows | winget no instala paquetes | Catálogo de origen roto e ID con mayúsculas |
| 16 | Nov 2026 | Navegador | Error de certificado | Punto final copiado en la dirección |
| 17 | Nov 2026 | Alertas | El webhook de Proxmox devuelve 404 | Secreto con el tema de ntfy mal guardado |
| 18 | Nov 2026 | Pi-hole | Aviso de NTP | Un LXC sin privilegios no puede cambiar la hora |
| 19 | 02/11/2026 | PC | Samsung Magician no clona al M.2 | El M.2 es un modelo OEM |
| 20 | 02/11/2026 | docker01 / DNS | docker01 no resuelve tras poner IP fija | dhcpcd reescribió `resolv.conf` sin DNS |
| 21 | 02/11/2026 | Docker / GLPI | GLPI se reinicia en bucle | *Bind mount* creado como root; el contenedor usa uid 33 |
| 22 | 03/11/2026 | Docker / Paperless | Paperless falla al azar con `File not found` | `docker compose run` dejó una segunda instancia |
| 23 | 04–07/11/2026 | Proxmox / DNS | Pi-hole y Kuma parados 3 días sin aviso | Sin arranque automático y pve01 dependía de Pi-hole |
| 24 | 08/11/2026 | Copias | La copia diaria solo guardaba kuma01 | Tarea creada con selección manual de una sola máquina |
| 25 | 09/11/2026 | n8n | El chat público del agente rechaza la conexión | Redirección de la autenticación n8n User Auth |
| 26 | 13/11/2026 | Google Cloud / rclone | Bloqueo `403: access_denied` al autorizar rclone | App OAuth en modo prueba sin usuarios autorizados ni URLs de marca |
| 27 | 13/11/2026 | Copias / ntfy | El script sube a Drive pero no llega el aviso a ntfy | Barras invertidas en `grep` con `pipefail` y marcador del tema sin sustituir |

---

## Fases 1, 1b y 2 (Hyper-V)

### Incidencia 1 — La VM no arranca
- **Síntoma:** "El hipervisor no está en ejecución" al encender la VM.
- **Causa:** virtualización (SVM) desactivada en la BIOS.
- **Solución:** activar SVM en la BIOS.
- **Aprendizaje:** tras tocar la BIOS, apagado completo, no reinicio.

### Incidencia 2 — "Permission denied" con la contraseña correcta
- **Causa:** sintaxis incorrecta del comando ssh.
- **Aprendizaje:** leer bien la sintaxis antes de dudar de la contraseña.

### Incidencia 3 — SSH figura como "inactive (dead)" pero funciona
- **Causa:** activación por socket de systemd (`TriggeredBy: ssh.socket`): el servicio arranca solo cuando llega una conexión.
- **Aprendizaje:** comprobar los puertos en escucha con `ss -tln`, no solo el estado del servicio.

### Incidencia 4 — Archivo bloqueado por un nano huérfano
- **Causa:** sesión SSH cortada sin cerrar el editor; el proceso de nano quedó abierto con el archivo.

### Incidencia 5 — El portapapeles de Hyper-V escribe caracteres incorrectos
- **Causa:** la consola de Hyper-V simula pulsaciones de teclado, y la distribución del teclado cambia los símbolos.
- **Aprendizaje:** teclado en inglés de EE. UU. en servidores.

### Incidencia 6 — Pérdida de configuración al revertir un punto de control
- **Causa:** puntos de control automáticos de Hyper-V.
- **Aprendizaje:** desactivarlos; solo instantáneas deliberadas.

### Incidencia 7 — El endurecimiento de SSH no surte efecto
- **Causa:** en `sshd_config.d/` gana el **primer** archivo que define un valor, y el de cloud-init tenía un número menor que el mío.
- **Solución:** renombrar a `01-hardening.conf`.
- **Aprendizaje:** verificar la configuración efectiva con `sshd -T` y en los registros (*Accepted publickey* frente a *password*), no el archivo escrito.

### Incidencia 8 — Tarea de cron fuera de hora
- **Causa:** servidor en UTC frente a hora local.
- **Aprendizaje:** revisar la zona horaria y la hora de los registros.

### Incidencia 9 — El DC conserva su nombre automático
- **Causa:** el renombrado se perdió en una reinstalación y el DC se promovió con el nombre aleatorio.
- **Solución:** renombrar el DC con `netdom`.
- **Aprendizaje:** nombre, IP y zona horaria **antes** de promover.

### Incidencia 10 — El cliente no resuelve el dominio ni llega al DC
- **Causa:** el DC perdió la IP fija y pasó a DHCP (.46).
- **Aprendizaje:** "Destination host unreachable" desde la propia IP significa fallo de ARP: el destino no está en esa IP.

### Incidencia 11 — Registros DNS duplicados hacia una IP muerta
- **Causa:** el DC no retira sus registros antiguos al cambiar de IP.
- **Solución:** borrarlos con `-RecordData`, revisando también `_msdcs`.

### Incidencia 12 — VM congelada en el arranque
- **Causa:** fusión de un punto de control en segundo plano.
- **Aprendizaje:** no apagar durante la fusión.

---

## Fase 3 y Fase A (Proxmox)

### Incidencia 13 — El router no responde al DNS de pve01
- **Síntoma:** `Could not resolve host: tailscale.com`.
- **Diagnóstico:** escalera de red: IP, ruta, ping al router y ping a 1.1.1.1 correctos; falla solo la resolución de nombres.
- **Causa:** el único DNS de pve01 era el router (192.168.1.1). **Causa raíz confirmada el 01/11/2026:** el router Movistar no actúa como servidor DNS para la red; a los clientes DHCP les reparte directamente los DNS de Movistar (80.58.61.250/254). Por eso solo fallaba en el equipo con IP y DNS fijos.
- **Solución:** 1.1.1.1 como DNS principal y el router como secundario.
- **Aprendizaje:** "no hay internet" muchas veces significa "no hay DNS". El primer "sistema actualizado" era falso porque se comprobó con el DNS roto (después aparecieron 191 actualizaciones pendientes).

### Incidencia 14 — El pegado falla en la consola web de Proxmox
- **Síntoma:** aparecen `^[[200~` y `~` alrededor del texto pegado; bash no encuentra el comando.
- **Causa:** modo de pegado entre corchetes (*bracketed paste*) de bash, mal interpretado por la consola web.
- **Solución:** `bind 'set enable-bracketed-paste off'` y, de forma permanente, en `~/.inputrc`. Mejor aún: trabajar por SSH.

### Incidencia 15 — winget no puede instalar paquetes
- **Síntoma:** error `0x8a15000f`. `winget source update` decía "Listo" con las barras al 0 %.
- **Solución:** instalar el catálogo a mano con `Add-AppxPackage`. Después, la búsqueda fallaba porque el identificador es `Tailscale.Tailscale` (con mayúsculas) y `-e` exige coincidencia exacta. Un error de red posterior (`0x80072eff`) era un corte de conexión.
- **Aprendizaje:** "Listo" no significa que haya funcionado (regla 1). Buscar el ID con `winget search` antes de instalar.

### Incidencia 16 — Error de certificado al abrir una web
- **Síntoma:** `ERR_CERT_COMMON_NAME_INVALID` en `healthchecks.io.`
- **Causa:** se copió el punto final de una frase; el certificado no cubre el nombre con punto.
- **Solución:** escribir la dirección correctamente.
- **Aprendizaje:** nunca saltarse ese aviso. (Mismo origen que `apt install npm.`: copiar un punto pegado al comando.)

### Incidencia 17 — El webhook de Proxmox devuelve 404
- **Síntoma:** `http status: 404` al probar el destino ntfy.
- **Causa:** el valor del secreto con el tema no se guardó bien, así que la URL final no apuntaba al tema. El encabezado `Title` tenía el texto literal `title`.
- **Solución:** reescribir el secreto y poner `{{ title }}` en el encabezado. Enlazar el destino al `default-matcher`.
- **Aprendizaje:** probar cada vía de alerta con un evento real.

### Incidencia 18 — Aviso de NTP en Pi-hole
- **Síntoma:** "Error al ajustar la hora durante la sincronización NTP: Permisos insuficientes".
- **Causa:** Pi-hole v6 intenta ajustar el reloj del sistema, pero un LXC sin privilegios comparte el reloj del anfitrión y no puede cambiarlo.
- **Solución:** `pihole-FTL --config ntp.sync.active false` y borrar el mensaje.
- **Aprendizaje:** un contenedor comparte con el anfitrión el núcleo **y** el reloj.

### Incidencia 19 — Samsung Magician no clona al M.2
- **Síntoma:** "Esta unidad no es compatible" en Data Migration.
- **Causa:** el M.2 es un Samsung **OEM** (PM981, MZVLB256HBHQ-000L7); Magician solo admite modelos de venta al público.
- **Solución:** clonación disco a disco con **Rescuezilla** desde el USB de Ventoy, identificando origen y destino por modelo y tamaño. Tras clonar, Windows arrancó desde el Samsung y desactivó el Kingston por **colisión de firmas** (comportamiento esperado).
- **Aprendizaje:** tras clonar, el primer arranque debe hacerse sin el disco original, o comprobando en Administración de discos en qué disco está C:.

### Incidencia 20 — docker01 no resuelve nombres tras poner IP fija
- **Síntoma:** después de configurar la IP fija y reiniciar, docker01 no resolvía nombres.
- **Causa:** al reiniciar con la interfaz todavía en DHCP, dhcpcd liberó la concesión y reescribió `/etc/resolv.conf` sin servidores DNS.
- **Solución:** IP fija en `/etc/network/interfaces` (`ens18`) y DNS `1.1.1.1` / `1.0.0.1` en `/etc/resolv.conf`; comprobado después de un reinicio.
- **Aprendizaje:** tras cambiar la red, verificar después de un reinicio, no solo en caliente.

### Incidencia 21 — El contenedor de GLPI se reinicia en bucle
- **Síntoma:** el contenedor de GLPI entraba en un ciclo de reinicios.
- **Causa:** Docker crea como root (0:0) las carpetas de un *bind mount* que no existen, pero el contenedor corre con uid 33 (www-data) y no podía escribir. Los reinicios dejaron además una instalación a medias en la base de datos.
- **Solución:** `storage/glpi` con dueño 33:33 y la instalación rehecha desde cero.
- **Aprendizaje:** comprobar el UID del contenedor antes de usar *bind mounts*. Una instalación interrumpida se rehace, no se parchea.

### Incidencia 22 — Paperless-ngx falla al azar con `File not found`
- **Síntoma:** errores aleatorios de `File not found` al procesar documentos.
- **Causa:** `docker compose run ... createsuperuser` había arrancado una **segunda** instancia completa de Paperless conectada a la misma cola, que buscaba los archivos en su propio `/tmp` vacío.
- **Solución:** eliminar la instancia sobrante y usar `docker compose exec` para los comandos de administración.
- **Aprendizaje:** `run` crea un contenedor nuevo; `exec` entra en el que ya existe. Si un error no aparece donde lo esperas, busca qué otro proceso lo genera.

### Incidencia 23 — Pi-hole y Uptime Kuma parados 3 días sin aviso; pve01 dependía de Pi-hole para su DNS
- **Síntoma:** los dispositivos pierden la red al conectarse a Tailscale. En Proxmox, kuma01 y pihole01 aparecen detenidos.
- **Diagnóstico:** `last -x reboot` mostró varias caídas sin apagado ordenado (30/10, 01/11 y tres el 04/11 entre las 12:51 y las 13:10). `journalctl -b -1` terminaba con actividad normal y se cortaba de golpe, sin errores de hardware → corte de luz o apagado brusco. Temperaturas en reposo normales (26–32 °C con Glances). En ese registro, `tailscaled` reenviaba el DNS de pve01 a Pi-hole (192.168.1.244) con `context deadline exceeded`, y el aviso a ntfy fallaba con `timeout`.
- **Causa:** (1) los contenedores no tenían "Start at boot", así que tras el reinicio del 04/11 se quedaron parados, y Uptime Kuma no pudo avisar porque era él el que estaba caído. (2) Tailscale aceptaba el DNS de la tailnet en pve01, así que el anfitrión resolvía nombres a través de Pi-hole, un servicio que aloja él mismo. Sin Pi-hole, pve01 no resolvía `ntfy.sh` y las alertas no salían.
- **Solución:** `pct set 100 --onboot 1` y `pct set 101 --onboot 1`; `tailscale set --accept-dns=false` en pve01 (vuelve a usar 1.1.1.1); prueba del destino ntfy desde Proxmox.
- **Aprendizaje:** el vigilante no puede vivir dentro de lo que vigila, y la infraestructura base no puede depender de lo que aloja. Tailscale en Linux acepta el DNS de la tailnet por defecto: hay que desactivarlo en servidores con DNS fijo. Tras un reinicio, comprobar que **todo** volvió, no solo el anfitrión.
- **Hallazgo adicional:** el destino `mail-to-root` de Proxmox **nunca ha entregado nada**: Gmail rechaza con `550 5.7.1` el correo enviado directamente desde una IP doméstica. El único aviso real es ntfy.

### Incidencia 24 — La copia diaria solo guardaba kuma01
- **Síntoma:** la notificación de la copia de las 03:30 del 08/11 solo lista el VMID 100 (kuma01, 604 MB).
- **Diagnóstico:** se leyó el detalle de la notificación en lugar de quedarse con "backup successful". `ls -lh /var/lib/vz/dump` confirmó copias diarias solo de kuma01 desde el 01/11. La línea de la orden mostraba `vzdump 100 ...`.
- **Causa:** la tarea se creó cuando solo existía kuma01, con selección manual. pihole01 (CT 101) y docker01 (VM 103, con Paperless y n8n) nunca se añadieron, así que **no tenían copias automáticas**. La prueba de restauración de docker01 se hizo con una copia manual, lo que ocultó el fallo.
- **Solución:** añadir 101 y 103 a la tarea y lanzarla con *Run now*: salen 100, 101 y 103 en estado `ok` (copia completa ≈ 6 GB: 0,6 + 0,3 + 5,1 GB, 2 min). Espacio comprobado con `df -h /` (`root` de 64 GB al 20 %). Con 7 diarias + 2 semanales no cabía, así que la retención se bajó **temporalmente a 3 diarias + 1 semanal**, hasta llevar las copias al HDD. En la Fase A se cambió a **Modo de selección: Todo** (`all 1` en `/etc/pve/jobs.cfg`).
- **Aprendizaje:** "backup successful" significa que lo seleccionado se copió bien, no que se copió todo (regla 1: leer la salida literal). Usar el modo **Todo** para que entren también las máquinas futuras, revisar qué máquinas aparecen en la notificación cada vez que se crea una nueva, y calcular el espacio antes de ampliar lo que se copia. El informe semanal de n8n con Gemini detectó el problema solo al revisar las tareas `vzdump` de la semana.

### Incidencia 25 — El chat público del agente rechaza la conexión
- **Síntoma:** al abrir la URL pública del chat del agente (`http://n8n.home.arpa/webhook/.../chat`), el navegador muestra "n8n.home.arpa ha rechazado la conexión". El editor de n8n sí abre por nombre.
- **Diagnóstico:** escalera de capas. DNS correcto (Pi-hole: `n8n.home.arpa` → `.248`); proxy correcto (`curl -sI -H "Host: n8n.home.arpa" http://192.168.1.248/` → `200 OK`, `X-Served-By: n8n.home.arpa`); flujo publicado; `N8N_WEBHOOK_URL` correcto en `compose.yaml`. La página llegaba a cargar (pestaña "Chat" con el icono de n8n y aviso de guardar contraseña): el fallo ocurría después del inicio de sesión.
- **Causa:** la autenticación **n8n User Auth** del chat exige una sesión del editor válida para ese nombre y redirige al inicio de sesión; la redirección de vuelta apuntaba a una dirección que no aceptaba conexiones. *(La URL exacta de la redirección no se llegó a capturar.)*
- **Solución:** cambiar la autenticación del Chat Trigger a **Basic Auth**, con una credencial propia independiente de la cuenta de n8n, y volver a publicar. El chat funciona desde el PC y desde el Pixel con Tailscale.
- **Aprendizaje:** "conexión rechazada" con el DNS y el proxy verificados apunta a una **redirección**, no a la red: hay que mirar la URL final, no la que escribiste. Con varios nombres para el mismo servicio (IP:puerto y nombre), las cookies de sesión no se comparten entre ellos.

### Incidencia 26 — Bloqueo `403: access_denied` al autorizar rclone con Google Drive
- **Síntoma:** al abrir el enlace de `rclone authorize "drive"` en el navegador del PC usando el *Client ID* propio, Google bloquea el inicio de sesión con `Error 403: access_denied` indicando que *"RCLONE no ha completado el proceso de verificación de Google"*. Al ir a Google Auth Platform para publicar la app, el botón **Publicar app** aparece deshabilitado.
- **Diagnóstico:** lectura literal de los dos mensajes en pantalla (regla 1): el error 403 indica que *"la app se está probando y solo pueden acceder a ella los testers aprobados"*, y el recuadro emergente del botón deshabilitado indica que faltan las URL de página principal y política de privacidad en la sección de marca.
- **Causa:** un proyecto OAuth externo nuevo en Google Auth Platform nace en estado **Prueba** con la lista de usuarios de prueba vacía (bloqueando incluso al propio creador del proyecto) y no permite pasar a estado **Producción** hasta rellenar los campos obligatorios de marca y dominio autorizado. Además, al cerrarse el proceso `rclone authorize` tras el primer fallo, recargar la pestaña devolvió `ERR_CONNECTION_REFUSED` en `127.0.0.1`.
- **Solución:** rellenar en **Información de la marca** las URL requeridas (apuntando al perfil de GitHub) y el dominio autorizado (`github.com`), pasar la aplicación a **Producción** en **Público** (lo que evita además que el token caduque a los 7 días) y relanzar `rclone authorize` en la PowerShell del PC para que vuelva a escuchar en `127.0.0.1`.
- **Aprendizaje:** ser el propietario del proyecto en Google Cloud no te incluye automáticamente como usuario autorizado en una pantalla OAuth en modo de prueba; y para tareas desatendidas por `systemd`, publicar la app en producción evita que el token de refresco caduque cada semana.

### Incidencia 27 — El script `backup-to-drive.sh` sube las copias pero no envía la notificación a ntfy
- **Síntoma:** la primera ejecución manual de `/usr/local/bin/backup-to-drive.sh` sube los 10 archivos (18,21 GiB) a Google Drive y devuelve el prompt, pero no llega ninguna notificación al móvil.
- **Diagnóstico:** revisión del registro `/var/log/rclone-backup.log` (regla 4) y del código del script `/usr/local/bin/backup-to-drive.sh` (regla 3).
- **Causa:** dos problemas combinados en el bloque de notificación: (1) la expresión `grep -o '\"count\":[0-9]*'` buscaba literalmente barras invertidas en la salida JSON de `rclone size`, fallando al estar activo `set -euo pipefail`, y (2) la variable `NTFY_URL` todavía conservaba el marcador de posición en vez del tema real.
- **Solución:** sustituir el patrón por `grep -o '"count":[0-9]*'`, poner el tema real en `NTFY_URL` y volver a ejecutar `/usr/local/bin/backup-to-drive.sh`. La comprobación tardó segundos y entregó la notificación con `"count":10`.
- **Aprendizaje:** con `set -euo pipefail`, cualquier comando dentro de una sustitución `$(...)` que devuelva código distinto de cero detiene el script antes de llegar a la línea siguiente; siempre hay que probar la notificación de extremo a extremo.

---

## Otros aprendizajes de la fase 3 y fase A

- Una alerta o una copia que nunca se ha probado con un fallo real no está probada.
- El vigilante no puede vivir dentro de lo que vigila (Uptime Kuma frente a Healthchecks).
- No se toca la red ni el cortafuegos de un servidor en remoto sin una vía de emergencia.
- Uptime Kuma avisa en los **cambios de estado**, no repite mientras algo sigue caído.
- Las webs de descargas o *streaming* piratas son el peor caso para un bloqueador DNS y una fuente de redirecciones maliciosas: no aceptar notificaciones ni instalar apps desde ellas.
- Al usar el traductor del navegador en paneles técnicos, se traducen nombres de dominio y archivos: desactivarlo para diagnosticar.
- Un proceso que termina con `status=0/SUCCESS` no se reinicia con `Restart=on-failure`: para servicios que deben estar siempre vivos, `Restart=always`. Antes de arrancar un servicio, comprobar con `ss -tlnp` que el puerto está libre (Glances, 07/11).
- En un servicio con muchos hilos de datos, separar dónde se pulsa (nombres, para el usuario) de dónde se comprueba (IP, para el contenedor que no usa el mismo DNS).
- Mínimo privilegio también para las integraciones y la IA: cada servicio con su propia credencial de solo lectura, revocable por separado; el agente de IA solo tiene herramientas de lectura.
- La IA resume, pero no sustituye la fuente: el informe semanal acertó el problema de las copias y aun así se equivocó en un día. Los datos exactos se comprueban en Proxmox.
- En n8n, un nodo HTTP con método **GET** hacia ntfy responde con éxito pero no publica nada: verde no significa enviado.
- Para dar herramientas a un AI Agent de n8n hay que usar la variante **Tool** del nodo (se conecta debajo del agente), no el nodo normal de la cadena.
- En archivos YAML (como `services.yaml` de Homepage), comentar solo la primera línea de un bloque deja las claves hijas huérfanas y rompe la sintaxis: hay que comentar todas las líneas del bloque.
