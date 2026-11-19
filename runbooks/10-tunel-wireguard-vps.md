# Runbook 10 — Túnel WireGuard saliente hacia un VPS (fase I)

> **Objetivo:** publicar un servicio de Bastión en internet **sin abrir ningún puerto en el router de casa**. Un contenedor de casa (`edge01`) abre un túnel WireGuard *hacia fuera* contra una VM en Google Cloud (`bastion-edge`); la VM recibe el tráfico público con Caddy (HTTPS automático) y lo reenvía por el túnel.
> **Decisión (fase I):** túnel saliente propio con WireGuard y Caddy. **Cloudflare descartado por ahora** (el tráfico pasaría por un tercero). La regla «la casa nunca se expone» sigue vigente: el router no cambia.
> **Dónde:** `pve01` (crear el CT 105), `edge01` y `bastion-edge`. **Tiempo:** una sesión.
> ⚠️ Este documento es público: **no contiene claves privadas ni IP públicas**. Se usan marcadores (`<CLAVE_PRIVADA_VPS>`, `<IP_PUBLICA_VPS>`…). Las claves viven solo en cada máquina, con permisos 600.
> ⚠️ Los pasos de Proxmox y de la VM los ejecuta Martín; un agente solo prepara y explica los comandos.

---

## Arquitectura

```
Internet ──► bastion-edge (Google Cloud, Madrid, e2-micro)
              Caddy :80/:443  ──►  10.66.0.1 (wg0)
                                       ║  túnel WireGuard (UDP 51820)
                                       ║  lo abre edge01 hacia fuera
                              10.66.0.2 (wg0) ◄── edge01 (CT 105, 192.168.1.240)
                                       └──► servicio de prueba (10.66.0.2:8080)
```

| Pieza | Valor |
|---|---|
| Red del túnel | `10.66.0.0/24` |
| `bastion-edge` (VPS) | `10.66.0.1`, escucha UDP `51820` |
| `edge01` (CT 105) | `10.66.0.2`, IP de casa `192.168.1.240`, **sin puertos entrantes** |
| Subdominio de prueba | `prueba.martindeprada.dev` → IP pública de `bastion-edge` |

**Por qué `edge01` inicia la conexión:** en casa nada escucha desde internet. WireGuard con `PersistentKeepalive` mantiene abierta la salida aunque la IP de casa cambie, y por eso el VPS no necesita saber la IP de casa.

**Qué limita el daño si el VPS se compromete:** `edge01` no reenvía tráfico (`ip_forward=0`) y su cortafuegos solo deja entrar por `wg0` el puerto del servicio de prueba. Desde el túnel no hay ruta hacia el resto de la red de casa.

---

## 0. Antes de empezar

1. **Copia fresca** de `pve01` desde Proxmox (el CT 105 aún no existe, así que basta con el estado actual).
2. **Dominio:** en el DNS de `martindeprada.dev`, crea el registro `A` de `prueba` apuntando a la IP pública de `bastion-edge` (el paso 1 la reserva). Pon un TTL bajo (300 s).
3. **Vía de vuelta atrás:** léela ahora, está en la sección 8. Apagar `edge01` o ejecutar `wg-quick down wg0` corta todo al instante.

---

## 1. La VM `bastion-edge` en Google Cloud

1. **Región:** Madrid (`europe-southwest1`). **Tipo:** `e2-micro`. **SO:** Debian 13.
2. **IP externa estática reservada** (si no, cambia al reiniciar y se rompen el DNS y el túnel). Anótala solo en un gestor de contraseñas, no aquí.
3. **Reglas del cortafuegos de Google Cloud** (entrada): `tcp:80`, `tcp:443` y `udp:51820` desde cualquier origen (la IP de casa cambia). **SSH solo desde IAP**, el servicio de túneles de Google, cuyo rango es `35.235.240.0/20`. Nunca abras el 22 a todo internet:

```
gcloud compute firewall-rules create bastion-edge-ssh-iap \
  --direction=INGRESS --action=ALLOW --rules=tcp:22 \
  --source-ranges=35.235.240.0/20 --target-tags=bastion-edge
gcloud compute firewall-rules create bastion-edge-publico \
  --direction=INGRESS --action=ALLOW --rules=tcp:80,tcp:443,udp:51820 \
  --source-ranges=0.0.0.0/0 --target-tags=bastion-edge
```

   - Las reglas se aplican a la VM mediante la etiqueta de red `bastion-edge`. Si existe la regla `default-allow-ssh` del proyecto, **bórrala o desactívala** para que el SSH solo entre por IAP.
   - Para entrar: `gcloud compute ssh bastion-edge --tunnel-through-iap` (necesita el permiso *IAP-secured Tunnel User*).
4. **Presupuesto:** antes de crearla, calcula el coste en la calculadora de Google Cloud; los precios cambian y lo gratuito depende de la región. Mantén la alerta del presupuesto (fase D1).
5. Endurecimiento básico, igual que docker01: usuario con `sudo`, solo clave Ed25519, sin acceso de root ni contraseñas, y verificar con `sudo sshd -T`.

Instala los paquetes:

```
sudo apt update && sudo apt install -y wireguard caddy nftables
```

- `wireguard` trae `wg` y `wg-quick`; `caddy` es el servidor web con HTTPS automático.

---

## 2. El contenedor `edge01` (CT 105)

En **pve01**:

```
pct create 105 local:vztmpl/<PLANTILLA-DEBIAN-13>.tar.zst \
  --hostname edge01 --cores 1 --memory 256 --swap 256 \
  --rootfs local-lvm:2 --unprivileged 1 --onboot 1 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.240/24,gw=192.168.1.1 \
  --nameserver 1.1.1.1 --password
```

- Sin *nesting* ni privilegios: solo hace falta WireGuard y un servicio pequeño.
- `--nameserver 1.1.1.1`: la infraestructura base no depende de un DNS que ella misma aloja.
- `--password`: te pedirá la contraseña de root del contenedor (al gestor).

WireGuard en un LXC sin privilegios necesita el dispositivo `/dev/net/tun` y el módulo `wireguard` cargado **en el anfitrión**. Añade a `/etc/pve/lxc/105.conf`:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Comprueba el módulo en `pve01`, arranca y crea la instantánea:

```
modprobe wireguard && lsmod | grep wireguard
pct start 105 && pct snapshot 105 limpio
```

Dentro de `edge01`:

```
apt update && apt install -y wireguard nftables python3
```

---

## 3. Claves de WireGuard

En **cada** máquina (VPS y `edge01`), genera su par de claves. La privada no sale de ahí:

```
umask 077
wg genkey | tee /etc/wireguard/private.key | wg pubkey > /etc/wireguard/public.key
cat /etc/wireguard/public.key
```

- `umask 077` hace que los archivos nuevos solo los lea root.
- Copia **solo la clave pública** de cada lado al `wg0.conf` del otro. Nunca pegues una clave privada en el repositorio, en un chat ni en un ticket.

---

## 4. `wg0.conf` de los dos lados

### `bastion-edge` (VPS) — `/etc/wireguard/wg0.conf`

```ini
[Interface]
Address = 10.66.0.1/24
ListenPort = 51820
PrivateKey = <CLAVE_PRIVADA_VPS>

[Peer]
# edge01 (casa). Sin Endpoint: es edge01 quien inicia la conexión
PublicKey = <CLAVE_PUBLICA_EDGE01>
AllowedIPs = 10.66.0.2/32
```

### `edge01` (casa) — `/etc/wireguard/wg0.conf`

```ini
[Interface]
Address = 10.66.0.2/24
PrivateKey = <CLAVE_PRIVADA_EDGE01>

[Peer]
# bastion-edge (VPS)
PublicKey = <CLAVE_PUBLICA_VPS>
Endpoint = <IP_PUBLICA_VPS>:51820
AllowedIPs = 10.66.0.1/32
PersistentKeepalive = 25
```

- `AllowedIPs = 10.66.0.2/32` (en el VPS) y `10.66.0.1/32` (en casa): cada lado solo acepta tráfico de la IP del otro, nunca de la red de casa. **No uses `0.0.0.0/0`** en `edge01`: enviaría toda su salida por el VPS.
- `PersistentKeepalive = 25`: envía un paquete cada 25 s para que el router mantenga abierta la salida.

Protege el archivo y arranca el túnel en ambos:

```
chmod 600 /etc/wireguard/wg0.conf
systemctl enable --now wg-quick@wg0
```

Comprueba el túnel (primero arranca `edge01`, luego mira el VPS):

```
wg show
ping -c 3 10.66.0.1      # desde edge01
ping -c 3 10.66.0.2      # desde el VPS
```

En `wg show` debe aparecer `latest handshake` de hace unos segundos. Si no aparece, suele ser una clave pública cambiada, el puerto UDP cerrado en Google Cloud o el `Endpoint` mal escrito.

---

## 5. Cortafuegos `nftables` de `edge01`

`/etc/nftables.conf` en `edge01`:

```
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;

    ct state established,related accept
    ct state invalid drop
    iif "lo" accept
    icmp type echo-request accept

    # SSH solo desde la red de casa
    iif "eth0" ip saddr 192.168.1.0/24 tcp dport 22 accept

    # Servicio de prueba: solo desde el túnel y solo desde el VPS
    iif "wg0" ip saddr 10.66.0.1 tcp dport 8080 accept
  }

  chain forward {
    type filter hook forward priority 0; policy drop;
  }

  chain output {
    type filter hook output priority 0; policy accept;
  }
}
```

- `policy drop` en `input`: todo lo no permitido se descarta. Las respuestas del túnel saliente vuelven por `established`.
- `forward` con `drop`: aunque alguien llegue por el túnel, no puede saltar a la red de casa.

Aplica y activa **sin cerrar la sesión actual** hasta comprobar que puedes abrir otra:

```
nft -c -f /etc/nftables.conf     # solo comprueba la sintaxis
nft -f /etc/nftables.conf
systemctl enable --now nftables
sysctl net.ipv4.ip_forward       # debe ser 0
```

---

## 6. Servicio de prueba (usar y tirar) en `edge01`

Una página estática, para no exponer nada real:

```
mkdir -p /srv/prueba && echo "<h1>Bastión: túnel de prueba</h1>" > /srv/prueba/index.html
```

`/etc/systemd/system/prueba-web.service`:

```ini
[Unit]
Description=Servicio de prueba del tunel (usar y tirar)
After=wg-quick@wg0.service
Requires=wg-quick@wg0.service

[Service]
DynamicUser=yes
ExecStart=/usr/bin/python3 -m http.server 8080 --bind 10.66.0.2 --directory /srv/prueba
Restart=on-failure
ProtectSystem=strict
ProtectHome=yes
NoNewPrivileges=yes

[Install]
WantedBy=multi-user.target
```

- `--bind 10.66.0.2`: escucha **solo** en la IP del túnel, no en la red de casa.

```
systemctl daemon-reload && systemctl enable --now prueba-web
curl -s http://10.66.0.2:8080/        # desde el VPS
```

---

## 7. Caddy en `bastion-edge`

`/etc/caddy/Caddyfile`:

```
prueba.martindeprada.dev {
	reverse_proxy 10.66.0.2:8080

	header {
		Strict-Transport-Security "max-age=31536000"
		X-Content-Type-Options "nosniff"
		Referrer-Policy "no-referrer"
		-Server
	}
}
```

- Caddy obtiene y renueva solo el certificado HTTPS (necesita que el DNS ya apunte a la VM y que los puertos 80 y 443 estén abiertos en Google Cloud).
- `reverse_proxy` envía el tráfico a la IP del túnel.

```
caddy validate --config /etc/caddy/Caddyfile
systemctl reload caddy
curl -I https://prueba.martindeprada.dev
```

Debe responder `200` con la cabecera `Strict-Transport-Security`.

---

## 8. Plan de vuelta atrás (cerrar todo de inmediato)

| Qué quieres | Comando | Dónde |
|---|---|---|
| Cortar el túnel ya | `wg-quick down wg0` | `edge01` (o `pct stop 105` en `pve01`) |
| Dejar de servir en internet | `systemctl stop caddy` | `bastion-edge` |
| Quitar el subdominio | borrar el registro `A` de `prueba` | DNS del dominio |
| Desmontarlo todo | borrar la VM y el CT 105 | Google Cloud y `pve01` |

El router no se ha tocado en ningún paso, así que no hay nada que deshacer en casa.

---

## Verificación final de la fase I

- [ ] `wg show` en los dos lados muestra `latest handshake` reciente
- [ ] `https://prueba.martindeprada.dev` responde por HTTPS con el certificado válido
- [ ] Desde fuera, `edge01` no tiene ningún puerto abierto: solo salida
- [ ] En el router no hay reenvíos de puertos
- [ ] `ip_forward` es 0 en `edge01` y el `forward` de nftables está en `drop`
- [ ] Desde el VPS no se alcanza ninguna IP de `192.168.1.0/24` salvo la del túnel
- [ ] Reinicio de `edge01`: el túnel y el servicio de prueba vuelven solos
- [x] Monitor creado en Uptime Kuma con la URL de prueba, y aviso por ntfy si cae
- [ ] SSH a `bastion-edge` solo por IAP (`35.235.240.0/20`); el puerto 22 no responde desde internet
- [ ] Ninguna clave privada ni IP pública escrita en el repositorio
- [ ] Inventario, ROADMAP y plan maestro actualizados

**Limitaciones conocidas:** el VPS es un único punto de fallo y un tercero (Google) ve el tráfico antes de que entre al túnel; por eso solo se publica contenido que ya sería público. Para algo sensible hará falta autenticación delante.
