# Runbook 04 — Linux y Ansible (fase C)

> **Objetivo:** que cualquier máquina de Bastión, en casa o en la nube, se configure con un comando en lugar de a mano.
> **Regla de esta fase (tuya):** los comandos de las clases y los *playbooks* los escribes tú. Este runbook te da el entorno, un primer ejemplo completo y explicado, y los ejercicios. Lo que escribas tú es lo que podrás defender en una entrevista.

---

## 1. El contenedor de prácticas `lab01`

En **pve01**. Crea el contenedor (CT 102) con la plantilla de Debian 13 que ya tienes:

```
pct create 102 local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst \
  --hostname lab01 --cores 1 --memory 512 --swap 512 \
  --rootfs local-lvm:4 --unprivileged 1 --onboot 0 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.245/24,gw=192.168.1.1 \
  --nameserver 1.1.1.1 --password
```

- `pct create 102` crea el contenedor con ID 102 a partir de la plantilla.
- `--unprivileged 1`: sin privilegios sobre el anfitrión (más seguro).
- `--rootfs local-lvm:4`: disco de 4 GB en el SSD del sistema.
- `--net0 ...`: IP fija `.245` y puerta de enlace.
- `--onboot 0`: no arranca solo; es un laboratorio.
- `--password`: te pedirá la contraseña de root del contenedor (al gestor).

```
pct start 102 && pct snapshot 102 limpio
```

Lo arranca y crea la instantánea `limpio`. **Antes de cada clase**, vuelve al punto de partida:

```
pct rollback 102 limpio && pct start 102
```

Así puedes romper lo que quieras sin miedo. Para entrar: `pct enter 102`.

Las 12 clases están en `00-contexto-homelab.md`, sección 7. Método: concepto → comandos (los tecleas tú, sin copiar) → práctica → reto.

---

## 2. El nodo de control `ansible01`

**Qué es Ansible:** un programa que se conecta por SSH a tus máquinas y las deja en el estado que describes en un archivo de texto (*playbook*). No instala nada en las máquinas gestionadas. Si ejecutas el mismo playbook dos veces, la segunda no cambia nada: eso se llama **idempotencia**.

**1. Crea el contenedor** (igual que antes, con otros datos):

```
pct create 104 local:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst \
  --hostname ansible01 --cores 1 --memory 1024 --swap 512 \
  --rootfs local-lvm:6 --unprivileged 1 --onboot 1 \
  --net0 name=eth0,bridge=vmbr0,ip=192.168.1.246/24,gw=192.168.1.1 \
  --nameserver 1.1.1.1 --password
pct start 104 && pct enter 104
```

**2. Dentro de ansible01, crea tu usuario e instala Ansible:**

```
apt update && apt install -y sudo git ansible
adduser martin && usermod -aG sudo martin && su - martin
```

`adduser` crea el usuario, `usermod -aG sudo` le permite usar `sudo`, y `su - martin` cambia a él. Ansible no se ejecuta como root.

```
ansible --version
```

**3. Genera una clave SSH propia para Ansible:**

```
ssh-keygen -t ed25519 -C "ansible01" -f ~/.ssh/id_ed25519
```

Una clave por equipo, nunca copiada: si un día se compromete `ansible01`, revocas solo esta. Ponle contraseña a la clave.

**4. Autoriza esa clave en las máquinas gestionadas.** docker01 no acepta contraseñas, así que la clave se añade desde una sesión que ya tengas abierta en docker01:

```
cat ~/.ssh/id_ed25519.pub
```

(en ansible01: muestra la clave pública; cópiala)

```
echo "PEGA-AQUI-LA-CLAVE-PUBLICA" >> ~/.ssh/authorized_keys
```

(en docker01, como `martin`: la añade a las claves autorizadas)

Prueba desde ansible01: `ssh martin@192.168.1.248` debe entrar sin pedir contraseña de usuario.

---

## 3. El proyecto de Ansible

En ansible01, dentro de tu clon del repositorio (`git clone` de `bastion`), crea esta estructura en la carpeta `ansible/`:

```
ansible/
├── ansible.cfg
├── inventory.ini
├── playbooks/
│   └── actualizar.yml
└── roles/
```

**`ansible.cfg`** — ajustes del proyecto:

```ini
[defaults]
inventory = inventory.ini
remote_user = martin
host_key_checking = True
```

**`inventory.ini`** — la lista de máquinas, por grupos:

```ini
[docker]
docker01 ansible_host=192.168.1.248

[laboratorio]
lab01 ansible_host=192.168.1.245 ansible_user=root

[nube]
# bastion-web ansible_host=100.x.y.z   (IP de Tailscale, cuando exista)
```

**Primera prueba** — comando *ad hoc* (sin playbook):

```
ansible docker -m ping
```

`-m ping` usa el módulo `ping`, que comprueba que Ansible puede conectarse y ejecutar Python en la máquina. Debe responder `"ping": "pong"` en verde.

```
ansible docker -m command -a "uptime"
```

Ejecuta `uptime` en todas las máquinas del grupo `docker`.

---

## 4. Primer playbook, explicado línea a línea

`playbooks/actualizar.yml`:

```yaml
---
- name: Actualizar los sistemas Debian
  hosts: docker
  become: true

  tasks:
    - name: Actualizar la lista de paquetes y aplicar actualizaciones
      ansible.builtin.apt:
        update_cache: true
        upgrade: dist
        cache_valid_time: 3600

    - name: Comprobar si hace falta reiniciar
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: reinicio

    - name: Avisar si hace falta reiniciar
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} necesita reiniciarse"
      when: reinicio.stat.exists
```

- `hosts: docker` — a qué grupo del inventario se aplica.
- `become: true` — ejecuta las tareas con `sudo`.
- Cada tarea tiene un `name` (lo que verás en pantalla) y un **módulo** (`apt`, `stat`, `debug`).
- `register: reinicio` guarda el resultado de una tarea en una variable, y `when:` ejecuta la siguiente solo si se cumple la condición.

Ejecútalo primero en modo simulación:

```
ansible-playbook playbooks/actualizar.yml --check --ask-become-pass
```

`--check` muestra lo que cambiaría sin tocar nada; `--ask-become-pass` pide tu contraseña de `sudo`. Después, sin `--check`. Ejecútalo dos veces: la segunda debe salir todo en `ok` y nada en `changed` (idempotencia).

---

## 5. Tus ejercicios (los escribes tú)

Cada uno termina con un *commit* en GitHub.

| # | Ejercicio | Módulos que necesitarás | Cómo sabes que está bien |
|---|---|---|---|
| 1 | Playbook que instala `htop`, `curl` y `jq` en `lab01` | `apt` | La segunda ejecución no cambia nada |
| 2 | Crear un usuario con su clave SSH autorizada | `user`, `authorized_key` | Entras con ese usuario sin contraseña |
| 3 | **Endurecer SSH** con una plantilla de `01-hardening.conf` y un *handler* que recarga el servicio | `template`, `handlers`, `validate: sshd -t -f %s` | `sshd -T` muestra los valores; si rompes la plantilla, no se aplica |
| 4 | Instalar Docker desde el repositorio oficial | `apt_key`/`deb822_repository`, `apt`, `service` | `docker run hello-world` funciona |
| 5 | Convertir 2, 3 y 4 en un **rol `bootstrap`** | `roles/bootstrap/tasks/main.yml` | Una máquina recién creada queda como docker01 con un comando |
| 6 | Guardar un secreto con **Ansible Vault** | `ansible-vault create` | El archivo en GitHub es ilegible |
| 7 | Desplegar un servicio: copiar un `compose.yaml` y levantarlo | `copy`, `community.docker.docker_compose_v2` | El contenedor queda `Up` |

**El ejercicio 5 es la pieza clave del proyecto:** ese rol es el que se aplicará a la VM de Google Cloud (runbook 05), a la de Azure (runbook 08) y a la de Oracle (runbook 09). Es lo que convierte "migré entre nubes" en un comando.

Prueba siempre primero en `lab01` (con `pct rollback 102 limpio` entre intentos) y solo después en docker01.

---

## Verificación final de la fase C

- [ ] `ansible all -m ping` responde `pong` en todas las máquinas
- [ ] `actualizar.yml` es idempotente
- [ ] Rol `bootstrap` probado en `lab01` desde la instantánea `limpio`
- [ ] Carpeta `ansible/` en GitHub, sin secretos en claro
- [ ] Inventario: `lab01` (.245) y `ansible01` (.246, CT 104) en ✅
