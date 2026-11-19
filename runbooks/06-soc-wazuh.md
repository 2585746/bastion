# Runbook 06 — SOC con Wazuh (fase E)

> **Objetivo:** un centro de operaciones de seguridad a pequeña escala: un SIEM que recoge registros de todas las máquinas, detecta comportamientos sospechosos, responde bloqueando y te resume el día con IA.
> **Requisito:** 24 GB de RAM (fase B1, hecha).
> **Enfoque:** defensivo. Las "pruebas de ataque" de este runbook son acciones inofensivas y conocidas, hechas contra una máquina tuya y desechable, para comprobar que la detección funciona.
> **Versiones:** los números de versión cambian. Usa siempre los de la [documentación oficial de Wazuh para Docker](https://documentation.wazuh.com/current/deployment-options/docker/wazuh-container.html).

---

## Conceptos en 30 segundos

- **SIEM:** sistema que centraliza registros de muchas máquinas y los convierte en alertas.
- **Wazuh:** SIEM libre. Tiene tres piezas: el **manager** (analiza), el **indexer** (guarda y busca) y el **dashboard** (la web).
- **Agente:** programa pequeño en cada máquina vigilada, que envía sus registros al manager.
- **Respuesta activa:** acción automática ante una alerta, como bloquear una IP.

```
Máquinas de casa (agentes) ─┐
Web en la nube (agente) ────┼──► Wazuh manager ──► alertas ──► n8n + Gemini ──► ntfy
Honeypot en la nube ────────┘         │
                                      └──► respuesta activa (bloqueo de IP)
```

---

## 1. La VM `wazuh01`

En **pve01**. Recursos: 2 vCPU, 6 GB (mínimo 4), 50 GB en `local-lvm`, IP `192.168.1.249`.

Crea la VM igual que docker01 (Debian 13 netinst, QEMU agent, `onboot`), o mejor: **con tu rol `bootstrap` de Ansible**, que deja SSH endurecido y Docker instalado. Es la primera máquina "de verdad" que montas con tu automatización.

Dentro de wazuh01, un ajuste que el indexer necesita:

```
echo "vm.max_map_count=262144" | sudo tee /etc/sysctl.d/99-wazuh.conf && sudo sysctl --system
```

`vm.max_map_count` es un límite del núcleo de Linux que el motor de búsqueda de Wazuh necesita más alto. `tee` lo escribe en un archivo para que sobreviva a los reinicios y `sysctl --system` lo aplica ya.

---

## 2. Desplegar Wazuh con Docker

```
sudo mkdir -p /opt/wazuh && sudo chown $USER:$USER /opt/wazuh && cd /opt/wazuh
git clone https://github.com/wazuh/wazuh-docker.git -b vX.Y.Z
cd wazuh-docker/single-node
```

`git clone -b vX.Y.Z` descarga los archivos oficiales de una versión concreta: sustituye `vX.Y.Z` por la última estable de la documentación. `single-node` es el despliegue de un solo servidor.

**Limita la memoria del indexer** antes de arrancar. En `docker-compose.yml`, en el servicio `wazuh.indexer`, busca la variable de Java y déjala así:

```yaml
      - "OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g"
```

Reserva 1 GB para el motor de búsqueda. Suficiente para un laboratorio.

**Genera los certificados** con los que las tres piezas se hablan cifrado:

```
docker compose -f generate-indexer-certs.yml run --rm generator
```

**Arranca:**

```
docker compose up -d && docker compose ps
```

La primera vez tarda varios minutos. Los tres contenedores deben quedar `Up`.

**Cambia las contraseñas por defecto** siguiendo la página "Changing the default password" de la documentación oficial. Es obligatorio: las de fábrica son públicas. Guárdalas en el gestor.

Abre `https://192.168.1.249` (certificado autofirmado) y entra al panel.

**Integra con lo que ya tienes:**
- Pi-hole: registro `wazuh.home.arpa` → `.248`. NPM: proxy host a `https://192.168.1.249:443` con *websockets*.
- Uptime Kuma: monitor HTTPS (ignorando el certificado).
- Proxmox: `qm snapshot <id> wazuh-ok`.

---

## 3. Agentes en las máquinas

En cada máquina Debian a vigilar (docker01, pihole01, kuma01, pve01, la web de la nube):

```
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
```

Añade la clave y el repositorio oficial de Wazuh (ajusta `4.x` a la rama de tu versión).

```
sudo WAZUH_MANAGER="192.168.1.249" WAZUH_AGENT_NAME="docker01" apt install -y wazuh-agent
sudo systemctl enable --now wazuh-agent
```

Instala el agente indicándole dónde está el manager y con qué nombre registrarse; después lo activa. **La versión del agente no puede ser mayor que la del manager.**

En el panel → **Agents**: cada máquina debe aparecer `Active`.

> Este es un ejercicio perfecto para Ansible: un rol `wazuh_agent` que haga estos pasos en todas las máquinas del inventario.

Qué obtienes sin configurar nada más: intentos de acceso SSH, uso de `sudo`, cambios en archivos de `/etc` (integridad), paquetes con vulnerabilidades conocidas y evaluación de configuración segura (CIS).

---

## 4. La máquina de pruebas `objetivo01`

Una VM Debian mínima y **desechable** (1 vCPU, 1 GB, 8 GB de disco, IP `192.168.1.250`), con SSH por contraseña activado **a propósito** y el agente de Wazuh instalado.

**Aislamiento con cortafuegos** dentro de objetivo01:

```
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default deny outgoing
sudo ufw allow from IP-DE-KALI to any port 22 proto tcp
sudo ufw allow out to 192.168.1.249 port 1514,1515 proto tcp
sudo ufw allow out 53
sudo ufw enable && sudo ufw status verbose
```

- Por defecto no entra ni sale nada.
- Solo la VM de pruebas del PC puede conectarse, y solo al puerto 22.
- Solo puede salir hacia Wazuh (puertos 1514 y 1515 del agente) y DNS.

```
qm snapshot <id> limpio
```

(en pve01) Instantánea para volver al estado inicial después de cada sesión.

> **Límite de este diseño:** objetivo01 está en tu red de casa, protegida por su propio cortafuegos. Sirve para pruebas de detección inofensivas. **No instales en ella software vulnerable a propósito** (tipo Metasploitable o DVWA) hasta tener una red de laboratorio separada con OPNsense (puente interno `vmbr1` sin salida a la red de casa). Esa es la evolución natural de esta fase.

**La VM de pruebas en tu PC:** Kali Linux en Hyper-V o VirtualBox, en modo puente y con IP fija (la que pusiste en la regla de `ufw`). Encendida solo cuando practiques.

---

## 5. Pruebas de detección (contra objetivo01 únicamente)

Cada prueba sigue el mismo ciclo, que es el trabajo real de un analista: **hacer → buscar la alerta → entenderla → documentarla.**

| # | Prueba (inofensiva) | Cómo | Qué debes ver en Wazuh |
|---|---|---|---|
| 1 | Accesos fallidos | Desde Kali, intenta entrar por SSH con una contraseña equivocada 8–10 veces seguidas | Alertas de autenticación fallida y, al repetirse, una de **fuerza bruta** de nivel alto |
| 2 | Usuario inexistente | `ssh noexiste@192.168.1.250` | Alerta de intento con usuario no válido |
| 3 | Integridad de archivos | En objetivo01: `echo "# prueba" \| sudo tee -a /etc/hosts` | Alerta de archivo modificado, con qué cambió |
| 4 | Cuenta nueva | En objetivo01: `sudo useradd prueba-soc` | Alerta de creación de usuario |
| 5 | Escalada con sudo | En objetivo01: un `sudo` con contraseña incorrecta 3 veces | Alerta de fallo de `sudo` |

En el panel: **Threat Hunting → Events**, filtra por `agent.name: objetivo01`. Para cada alerta, anota: ID de la regla, nivel, descripción y la técnica **MITRE ATT&CK** que Wazuh le asigna (por ejemplo, fuerza bruta = T1110).

**Lo que Wazuh no ve por sí solo:** un escaneo de puertos. Para eso hace falta un detector de red (Suricata), que es la siguiente ampliación junto con OPNsense.

Al terminar: `qm rollback <id> limpio`.

---

## 6. Respuesta activa: bloquear automáticamente

Haz que Wazuh bloquee durante 10 minutos la IP que dispare la alerta de fuerza bruta SSH.

En wazuh01, edita la configuración del manager:

```
nano /opt/wazuh/wazuh-docker/single-node/config/wazuh_cluster/wazuh_manager.conf
```

Añade antes de `</ossec_config>`:

```xml
  <active-response>
    <command>firewall-drop</command>
    <location>local</location>
    <rules_id>5763</rules_id>
    <timeout>600</timeout>
  </active-response>
```

- `firewall-drop`: orden incluida en Wazuh que añade una regla de bloqueo.
- `location local`: se ejecuta en la máquina donde ocurrió el ataque.
- `rules_id 5763`: la regla de fuerza bruta SSH. **Confirma el ID** con el que viste en tu prueba 1.
- `timeout 600`: el bloqueo se retira solo a los 10 minutos.

```
cd /opt/wazuh/wazuh-docker/single-node && docker compose restart wazuh.manager
```

**Repite la prueba 1.** Esta vez, tras los intentos, Kali deja de poder conectarse. En objetivo01, `sudo iptables -L INPUT -n | head` muestra la IP bloqueada, y en el panel aparece la alerta de respuesta activa. A los 10 minutos vuelve el acceso.

> Cuidado con aplicarlo a tus máquinas reales: un despiste con tu propia contraseña te bloquearía a ti. Empieza solo con objetivo01.

---

## 7. Honeypot en la nube (ataques reales)

**Qué es:** un servicio SSH falso (Cowrie) expuesto a internet. No da acceso a nada real, pero registra quién se conecta, con qué contraseñas prueba y qué intenta ejecutar.

**Aislamiento, primero:**
- VM **separada** de la web: `bastion-honey`, `e2-micro` o `e2-small`, etiqueta `honeypot`.
- Etiqueta de Tailscale propia, que solo puede enviar registros a Wazuh. En la ACL:

```json
{ "action": "accept", "src": ["tag:honeypot"], "dst": ["192.168.1.249:1514", "192.168.1.249:1515"] }
```

- Únela con `sudo tailscale up --advertise-tags=tag:honeypot --accept-routes` (aquí sí hace falta `--accept-routes` para alcanzar Wazuh por tu router de subred; la ACL limita a esos dos puertos). **Sin** `--ssh`.
- Comprueba desde ella que **no** llega a nada más de tu casa, igual que en el runbook 05.

**Despliegue de Cowrie** (`/opt/cowrie/compose.yaml`):

```yaml
services:
  cowrie:
    image: cowrie/cowrie:latest
    container_name: cowrie
    restart: unless-stopped
    ports:
      - "22:2222"
    volumes:
      - cowrie_logs:/cowrie/cowrie-git/var/log/cowrie
    read_only: false
    cap_drop:
      - ALL

volumes:
  cowrie_logs:
```

Cowrie escucha en el 2222 y se publica en el **22** de la VM. Antes, mueve el SSH real de la VM a otro puerto (por ejemplo 22022) en `/etc/ssh/sshd_config.d/` y adminístrala por un túnel IAP de Google:

```
gcloud compute firewall-rules create honeypot-ssh-falso --allow=tcp:22 --target-tags=honeypot --source-ranges=0.0.0.0/0
gcloud compute firewall-rules create honeypot-admin-iap --allow=tcp:22022 --target-tags=honeypot --source-ranges=35.235.240.0/20
gcloud compute start-iap-tunnel bastion-honey 22022 --local-host-port=localhost:2222 --zone=europe-southwest1-a
```

- La primera regla abre el SSH **falso** a todo internet.
- La segunda permite el SSH **real** solo desde el rango de IAP de Google (`35.235.240.0/20`).
- La tercera abre un túnel: en otra ventana, `ssh -p 2222 usuario@localhost`.

**Envía los registros a Wazuh:** instala el agente en `bastion-honey` y añade en su `/var/ossec/etc/ossec.conf`:

```xml
  <localfile>
    <log_format>json</log_format>
    <location>/var/lib/docker/volumes/cowrie_cowrie_logs/_data/cowrie.json</location>
  </localfile>
```

Reinicia el agente. En pocas horas tendrás intentos reales de todo el mundo en tu panel.

---

## 8. Informe diario de amenazas con IA

Flujo en n8n: **Schedule (cada día 8:00) → HTTP Request al indexer → Code → Gemini → ntfy.**

**Consulta al indexer** (nodo HTTP Request): `POST https://192.168.1.249:9200/wazuh-alerts-*/_search`, autenticación básica con un usuario **de solo lectura** creado en el panel (Indexer management → Security), *Ignore SSL Issues* activado, y este cuerpo JSON:

```json
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "range": { "timestamp": { "gte": "now-24h" } } },
        { "range": { "rule.level": { "gte": 7 } } }
      ]
    }
  },
  "aggs": {
    "por_regla":  { "terms": { "field": "rule.description", "size": 10 } },
    "por_agente": { "terms": { "field": "agent.name", "size": 10 } },
    "por_origen": { "terms": { "field": "data.srcip", "size": 10 } },
    "por_tecnica": { "terms": { "field": "rule.mitre.id", "size": 10 } }
  }
}
```

Pide solo **resúmenes** de las últimas 24 horas (alertas de nivel 7 o más): las 10 reglas, máquinas, IP de origen y técnicas MITRE más frecuentes. Así a Gemini le llegan cifras agregadas, no registros en bruto.

**Prompt para Gemini:**

```
Eres analista de un SOC. Con los datos agregados de Wazuh de las últimas 24 horas, escribe un informe breve en español neutro, en texto plano, máximo 12 líneas.
Primera línea: "Sin incidentes relevantes" o "Requiere revisión".
Después: qué se intentó, contra qué máquina, desde dónde y con qué técnica MITRE ATT&CK.
Distingue el honeypot (ataques esperados) de las máquinas reales (lo importante).
No inventes datos ni recomiendes acciones que no se deduzcan de las cifras.

Datos:
{{ $json.resumen }}
```

**Reglas de datos:** las IP de atacantes y nombres de reglas no son datos personales tuyos, pero no envíes a Gemini nombres de usuario reales, rutas internas ni contenido de archivos.

**Amplía el agente de Bastión:** añade al AI Agent una herramienta `alertas_seguridad` con esta misma consulta, para preguntarle desde el móvil "¿hubo algo raro hoy?".

---

## Verificación final de la fase E

- [ ] Panel de Wazuh accesible, contraseñas por defecto cambiadas
- [ ] Todos los agentes `Active`
- [ ] Las 5 pruebas generan su alerta y están documentadas con su técnica MITRE
- [ ] La respuesta activa bloquea y desbloquea sola en objetivo01
- [ ] El honeypot no alcanza nada de casa salvo los puertos de Wazuh
- [ ] Llega el informe diario por ntfy
- [ ] Copias: wazuh01 entra en la tarea diaria (modo "Todo")
- [ ] Inventario (.249, .250, nube), incidencias y ROADMAP actualizados

**Para el portafolio:** un artículo por prueba ("Cómo detecta mi SOC un ataque de fuerza bruta"), con captura de la alerta, la regla y la respuesta. Es la sección más valiosa de la web.
