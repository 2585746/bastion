# Runbook 09 — Migración a Oracle Always Free y cierre de Google Cloud (fase H)

> **Objetivo:** que la web y el honeypot sigan funcionando gratis de forma permanente, y que Google Cloud no cobre nada al terminar la prueba.
> **Cuándo:** al menos **dos semanas antes** de la fecha de caducidad de los créditos (recordatorio en el calendario).
> **Orden:** primero levantar el destino, después cambiar el DNS, y solo al final borrar el origen.

---

## 1. Cuenta y VM en Oracle Cloud

1. Crea la cuenta en Oracle Cloud (Always Free). Pide tarjeta para verificar la identidad, pero los recursos *Always Free* no se cobran. Elige una región cercana (Madrid o Fráncfort): **no se puede cambiar después**.
2. Crea una instancia de cómputo dentro de los límites *Always Free* (el panel los marca con la etiqueta "Always Free-eligible"), con Debian o Ubuntu y tu clave pública SSH.
3. En la **Security List** de su subred, añade reglas de entrada para TCP 80 y 443 desde `0.0.0.0/0`.

> Las formas y cantidades gratuitas cambian: usa las que el panel marque como *Always Free* el día que la crees. Las instancias ARM gratuitas a veces no tienen capacidad disponible; si falla, reintenta más tarde o usa la forma x86 pequeña.

**Cortafuegos del propio sistema:** las imágenes de Oracle traen reglas de `iptables` que bloquean casi todo. Ábrelo también dentro de la VM:

```
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 443 -j ACCEPT
sudo netfilter-persistent save
```

Inserta reglas que aceptan conexiones nuevas a los puertos web y las guarda. Si la web no carga y la Security List está bien, casi siempre es esto.

---

## 2. Dejarla como las demás

Lo mismo que hiciste en Google y en Azure:

1. Tailscale con `tag:nube` (runbook 05, paso 3) y comprobación de que **no** alcanza tu casa.
2. Añádela al inventario y aplica el rol `bootstrap`:

```
ansible-playbook playbooks/bootstrap.yml --limit bastion-web-oci
```

`--limit` aplica el playbook solo a esa máquina.

3. Copia la configuración de la web desde tu repositorio:

```
scp -r compose/web/* martin@bastion-web-oci:/opt/web/
```

---

## 3. Mover la web sin corte

**1. Baja el TTL del DNS un día antes.** En el panel del dominio, pon el TTL de los registros A en 300 segundos. Así el cambio se propaga en 5 minutos en vez de horas.

**2. Sube el sitio a la VM nueva** (`npm run build` y `scp` de `dist/`, como en el runbook 05).

**3. Prueba antes de cambiar el DNS**, engañando solo a tu equipo:

```
curl -sI --resolve tudominio.com:80:IP-DE-ORACLE http://tudominio.com | head -n 1
```

`--resolve` hace que `curl` use la IP nueva para ese nombre sin tocar el DNS real. Debe responder (una redirección a HTTPS es buena señal).

**4. Cambia los registros A** (`@` y `www`) a la IP de Oracle.

**5. Comprueba la propagación y el certificado:**

```
dig +short tudominio.com
docker compose -f /opt/web/compose.yaml logs caddy | grep -i certificate
```

Cuando `dig` devuelva la IP nueva, Caddy pedirá el certificado solo. Abre `https://tudominio.com`.

**6. CrowdSec** en la VM nueva (runbook 05, paso 5).

**7. Uptime Kuma:** el monitor del dominio debe seguir en verde durante todo el proceso. Ese gráfico sin cortes es tu evidencia de una migración limpia.

---

## 4. Mover el honeypot

Repite el runbook 06, paso 7, en una segunda instancia gratuita (o en la misma si no hay capacidad, sabiendo que comparten máquina). Los registros antiguos ya están en Wazuh, en casa: no hay datos que migrar.

En Oracle no existe el túnel IAP de Google. Para administrar el SSH real de la VM del honeypot, limita su puerto en la Security List a **tu IP pública de casa** (`/32`), o usa el servicio Bastion de Oracle.

---

## 5. Lo que hay que rescatar de Google Cloud

| Dato | Dónde está | Qué hacer |
|---|---|---|
| Configuración (Compose, Caddyfile) | Ya en GitHub | Nada |
| Sitio web | Se regenera con `npm run build` | Nada |
| Registros de Caddy y CrowdSec | `/opt/web/logs` | Opcional: `scp` a casa si quieres conservarlos |
| Registros del honeypot | Ya enviados a Wazuh | Nada |
| Certificados | Caddy los pide de nuevo | Nada |

Si diseñaste bien, **no hay nada que rescatar**. Esa es la demostración de que la infraestructura era reproducible.

---

## 6. Cerrar Google Cloud

**Espera 48 horas** con todo funcionando en Oracle antes de borrar.

**1. Mira qué existe:**

```
gcloud compute instances list
gcloud compute addresses list
gcloud compute disks list
gcloud compute snapshots list
```

Las IP reservadas, los discos sueltos y las instantáneas siguen costando aunque no haya VM.

**2. Borra el proyecto entero** (por eso se creó un proyecto propio):

```
gcloud projects delete bastion-cloud-XXXX
```

Elimina el proyecto y todo su contenido. Se puede recuperar durante 30 días; después se borra definitivamente.

**3. No actualices la cuenta de facturación a pago** si no vas a seguir usando Google Cloud. Si la prueba termina sin actualizar, Google detiene los recursos y no cobra.

**4. Lo que no se borra con el proyecto:** la API de Gemini de AI Studio vive en su propio proyecto ("Proyecto de Gemini"). n8n depende de ella: **no la toques**. Revisa que su límite de gasto mensual sigue en 3 €.

**5. Comprueba la facturación una semana después:** Facturación → Informes. Debe estar a cero.

---

## 7. Retirar lo viejo de Tailscale y del inventario

- Consola de Tailscale → **Machines** → elimina `bastion-web` y `bastion-honey` de Google.
- Wazuh → **Agents** → elimina los agentes antiguos.
- Uptime Kuma → actualiza o borra monitores que apunten a IP antiguas.
- `inventory.ini` de Ansible → quita las máquinas de Google.

---

## Verificación final de la fase H

- [ ] `https://tudominio.com` servido desde Oracle, sin corte en la gráfica de Kuma
- [ ] Honeypot enviando registros a Wazuh desde Oracle
- [ ] `gcloud projects list` no muestra `bastion-cloud`
- [ ] Facturación de Google Cloud a cero una semana después
- [ ] La API de Gemini y n8n siguen funcionando (informe semanal del lunes)
- [ ] Máquinas antiguas fuera de Tailscale, Wazuh, Kuma y Ansible
- [ ] Inventario, ROADMAP y un artículo en el blog: "Migré mi infraestructura entre tres nubes"
