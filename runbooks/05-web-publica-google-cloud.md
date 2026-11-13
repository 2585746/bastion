# Runbook 05 — Web pública en Google Cloud (fase D)

> **Objetivo:** tu portafolio y blog en internet, con tu dominio, HTTPS y protección, en una VM de Google Cloud que se pueda mover a otra nube copiando una carpeta.
> **Requisitos:** el sitio web ya construido (ver `prompts/claude-code-portafolio.md`) y el dominio del GitHub Student Pack.
> **Dónde:** tu PC (con `gcloud`) y la VM nueva.
> **Los precios cambian:** antes de crear nada, calcula el coste en la calculadora de Google Cloud.

---

## Arquitectura

```
Visitante → dominio (DNS) → VM en Google Cloud (Madrid)
                              ├─ cortafuegos de Google: solo 80 y 443
                              ├─ Caddy: HTTPS automático + sitio estático
                              ├─ CrowdSec: bloquea IP maliciosas
                              └─ Tailscale: solo para administrar y enviar registros
Tu casa: ningún puerto abierto. La VM NO puede entrar a tu red.
```

---

## 0. Preparación (una vez)

**1. Fecha de caducidad.** Google Cloud → Facturación → Créditos: apunta la fecha exacta y **mueve el recordatorio del calendario** a dos semanas antes.

**2. Presupuesto** con alertas al 50, 90 y 100 % (ya creado). Recuerda: avisa, no frena.

**3. Instala `gcloud` en tu PC** (PowerShell):

```
winget install -e --id Google.CloudSDK
```

**4. Inicia sesión y crea el proyecto:**

```
gcloud auth login
gcloud projects create bastion-cloud-XXXX --name="Bastion Cloud"
gcloud config set project bastion-cloud-XXXX
```

El ID del proyecto debe ser único en todo Google: cambia `XXXX` por unos números. Tener un proyecto propio permite borrar todo de golpe al final.

```
gcloud billing projects link bastion-cloud-XXXX --billing-account=TU-ID-DE-FACTURACION
gcloud services enable compute.googleapis.com
```

Vincula el proyecto a la cuenta con los créditos (el ID sale en `gcloud billing accounts list`) y activa el servicio de máquinas virtuales.

---

## 1. Crear la VM

```
gcloud compute instances create bastion-web --zone=europe-southwest1-a --machine-type=e2-small --image-family=debian-13 --image-project=debian-cloud --boot-disk-size=20GB --tags=web
```

- `europe-southwest1-a`: Madrid.
- `e2-small`: 2 vCPU compartidas y 2 GB, suficiente para un sitio estático con CrowdSec. Sube a `e2-medium` (4 GB) si añades más servicios.
- `--tags=web`: etiqueta para aplicarle reglas de cortafuegos.

**Cortafuegos: solo web.**

```
gcloud compute firewall-rules create permitir-web --allow=tcp:80,tcp:443 --target-tags=web --source-ranges=0.0.0.0/0
```

Abre los puertos 80 y 443 solo en las máquinas con la etiqueta `web`.

```
gcloud compute firewall-rules list --format="table(name,sourceRanges.list(),allowed[].map().firewall_rule().list())"
```

Lista las reglas. La red por defecto trae una regla `default-allow-ssh` abierta a todo internet: la quitaremos en el paso 3, cuando Tailscale funcione.

**IP fija** (para que el dominio no se rompa si reinicias la VM):

```
gcloud compute addresses create bastion-web-ip --region=europe-southwest1
gcloud compute addresses describe bastion-web-ip --region=europe-southwest1 --format="value(address)"
```

Reserva una IP y la muestra. Asígnala:

```
gcloud compute instances delete-access-config bastion-web --zone=europe-southwest1-a --access-config-name="external-nat"
gcloud compute instances add-access-config bastion-web --zone=europe-southwest1-a --address=LA-IP-RESERVADA
```

> Una IP reservada **sin usar** se cobra. Al borrar la VM, libera también la IP.

---

## 2. Entrar y preparar la máquina

```
gcloud compute ssh bastion-web --zone=europe-southwest1-a
```

`gcloud` crea una clave y te conecta. Dentro de la VM:

```
sudo apt update && sudo apt full-upgrade -y
```

**Si ya tienes el rol `bootstrap` de Ansible (runbook 04):** añade la VM al inventario y aplícalo; es el momento de comprobar que sirve fuera de casa. Si no, instala Docker a mano:

```
curl -fsSL https://get.docker.com -o get-docker.sh
less get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
```

Descarga el instalador oficial, **léelo antes de ejecutarlo** (`less`), instálalo y añade tu usuario al grupo `docker` (cierra y abre la sesión para que surta efecto).

---

## 3. Tailscale, con la VM encerrada

**Riesgo:** tu tailnet tiene una ruta hacia toda la red de casa. Una máquina expuesta a internet no debe poder usarla.

**1. Define las reglas de acceso (ACL) ANTES de unir la VM.** Consola de Tailscale → **Access controls**. Añade una etiqueta y limita lo que puede hacer:

```json
{
  "tagOwners": {
    "tag:nube": ["autogroup:admin"]
  },
  "acls": [
    { "action": "accept", "src": ["autogroup:member"], "dst": ["*:*"] },
    { "action": "accept", "src": ["tag:nube"], "dst": ["192.168.1.248:5678"] }
  ]
}
```

- Tus dispositivos (`autogroup:member`) siguen llegando a todo, incluida la VM.
- Las máquinas con `tag:nube` solo pueden hablar con n8n (`.248:5678`), para enviar avisos. Nada más de tu casa.
- **Adapta este bloque a lo que ya tengas en tu ACL**; no borres reglas existentes sin entenderlas.

**2. Instala Tailscale en la VM y únela con la etiqueta:**

```
curl -fsSL https://tailscale.com/install.sh | sudo sh
sudo tailscale up --advertise-tags=tag:nube --ssh
```

`--advertise-tags` la une como `tag:nube`. `--ssh` activa el SSH de Tailscale: podrás entrar con `ssh usuario@bastion-web` desde tus dispositivos sin exponer el puerto 22. Para que funcione, la ACL necesita también una sección `ssh` que permita a tus dispositivos entrar en `tag:nube`:

```json
"ssh": [
  { "action": "check", "src": ["autogroup:member"], "dst": ["tag:nube"], "users": ["autogroup:nonroot"] }
]
```

`check` pide que te vuelvas a identificar cada cierto tiempo antes de abrir la sesión.

No se usa `--accept-routes`, así que la VM ni siquiera conoce la ruta hacia tu casa. Si más adelante necesita enviar avisos a n8n, se añade esa opción y la ACL limita el acceso a ese único puerto.

**3. Comprueba el encierro** desde la VM:

```
ping -c 2 192.168.1.242 ; curl -m 5 -sI http://192.168.1.248:81 | head -n 1
```

Ambos deben **fallar** (Proxmox y el panel de NPM no son alcanzables). Si responden, revisa la ACL antes de continuar.

**4. Cierra el SSH público:**

```
gcloud compute firewall-rules delete default-allow-ssh
```

(en tu PC) Desde ahora se administra por Tailscale. Si un día Tailscale falla, queda la consola serie de Google Cloud como emergencia.

---

## 4. Desplegar el sitio con Caddy

**Caddy** es un servidor web que consigue y renueva los certificados HTTPS solo.

En la VM:

```
sudo mkdir -p /opt/web/site && sudo chown -R $USER:$USER /opt/web && cd /opt/web
nano compose.yaml
```

```yaml
services:
  caddy:
    image: caddy:2
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./site:/srv:ro
      - ./logs:/var/log/caddy
      - caddy_data:/data
      - caddy_config:/config

volumes:
  caddy_data:
  caddy_config:
```

- `./site:/srv:ro`: tu web, en solo lectura.
- `caddy_data`: donde Caddy guarda los certificados (si se pierde, los vuelve a pedir).
- `./logs`: registros de acceso, que leerá CrowdSec.

```
nano Caddyfile
```

```
tudominio.com, www.tudominio.com {
    root * /srv
    encode zstd gzip
    file_server

    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        Referrer-Policy "strict-origin-when-cross-origin"
        -Server
    }

    log {
        output file /var/log/caddy/access.log
        format json
    }
}
```

- `file_server` sirve los archivos estáticos; `encode` los comprime.
- Las cabeceras `header` endurecen el navegador del visitante (HTTPS obligatorio, sin incrustar en otras webs).
- `-Server` oculta la versión del servidor.

**Apunta el dominio:** en el panel de tu dominio, crea dos registros **A** (`@` y `www`) con la IP reservada. Comprueba que ya resuelve:

```
dig +short tudominio.com
```

Cuando devuelva tu IP:

```
docker compose up -d && docker compose logs -f caddy
```

En los registros verás cómo obtiene el certificado (`certificate obtained successfully`). Sal con Ctrl+C.

**Sube el sitio.** En tu PC, dentro del proyecto de la web:

```
npm run build
scp -r dist/* usuario@bastion-web:/opt/web/site/
```

`npm run build` genera la carpeta `dist/` con el sitio y `scp` la copia por Tailscale. Abre `https://tudominio.com`.

> Mejora posterior: una acción de GitHub que construya y despliegue en cada `git push`.

---

## 5. CrowdSec: bloquear a quien ataca

**CrowdSec** lee los registros, detecta comportamientos maliciosos (escaneos, fuerza bruta) y bloquea esas IP. Además comparte una lista de IP maliciosas conocidas.

```
curl -s https://install.crowdsec.net | sudo sh
sudo apt install -y crowdsec crowdsec-firewall-bouncer-nftables
```

El primer comando añade el repositorio oficial; el segundo instala el motor y el *bouncer* (la pieza que bloquea en el cortafuegos).

```
sudo cscli collections install crowdsecurity/caddy crowdsecurity/linux
```

`cscli` es la herramienta de control. Las colecciones traen los analizadores y escenarios para Caddy y para Linux.

Dile dónde están los registros de Caddy:

```
sudo nano /etc/crowdsec/acquis.d/caddy.yaml
```

```yaml
filenames:
  - /opt/web/logs/access.log
labels:
  type: caddy
```

```
sudo systemctl restart crowdsec && sudo cscli metrics
```

En `Acquisition Metrics` debe aparecer tu archivo con líneas leídas. Tras unas horas en internet:

```
sudo cscli decisions list
sudo cscli alerts list
```

Verás IP bloqueadas y por qué. Esas alertas son material real para el SOC (runbook 06).

**WAF (opcional, después):** CrowdSec tiene un componente AppSec con reglas tipo OWASP. Requiere integrar el *bouncer* dentro de Caddy (una imagen de Caddy compilada con ese módulo). Hazlo cuando lo básico lleve días estable, siguiendo la documentación oficial de CrowdSec para Caddy.

---

## 6. Vigilancia y copia

- **Uptime Kuma (en casa):** monitor HTTP(s) a `https://tudominio.com` con aviso de caducidad del certificado.
- **Copia:** lo único con estado es `/opt/web` (Caddyfile, compose) y eso ya está en GitHub. El sitio se regenera con `npm run build`. Nada que perder.

---

## Verificación final de la fase D

- [ ] `https://tudominio.com` carga con candado válido
- [ ] Desde la VM **no** se llega a `192.168.1.242` ni al panel de NPM
- [ ] No existe la regla `default-allow-ssh`; se entra por Tailscale
- [ ] `cscli metrics` muestra registros de Caddy leídos
- [ ] Prueba externa de cabeceras y TLS (por ejemplo, SSL Labs) con buena nota
- [ ] `compose.yaml` y `Caddyfile` en GitHub (`compose/web/`)
- [ ] Inventario (nueva sección "Nube") y ROADMAP actualizados

**Para migrar a otra nube (runbook 09):** crear VM → rol `bootstrap` → copiar `/opt/web` → `docker compose up -d` → cambiar el registro A.
