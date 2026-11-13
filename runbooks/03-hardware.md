# Runbook 03 — Sesión de hardware (fase B)

> **Objetivo:** 24 GB de RAM, un SSD dedicado a las máquinas virtuales (`vmdata`) y un disco de 2 TB para las copias.
> **Dónde:** en Yepes, con el servidor delante. **Tiempo:** una tarde (Memtest86+ y el SMART largo tardan horas, pero corren solos).
> ⚠️ Los comandos de discos **borran datos**. Identifica cada disco por modelo y tamaño antes de ejecutar nada.

---

## 0. Antes de apagar

**1. Copia fresca de todo** (Proxmox → Centro de datos → Respaldo → Ejecutar ahora) y comprueba que la copia a Drive terminó:

```
systemctl start backup-to-drive.service && tail -n 5 /var/log/rclone-backup.log
```

Lanza la subida a mano y muestra las últimas 5 líneas del registro.

**2. Anota el estado actual de los discos:**

```
lsblk -o NAME,SIZE,MODEL,SERIAL,MOUNTPOINT
```

`lsblk` lista los discos; `-o` elige las columnas: nombre, tamaño, modelo, número de serie y dónde está montado. Guarda una captura: te servirá para distinguir los discos nuevos.

**3. Apaga ordenadamente:**

```
shutdown -h now
```

Desenchufa el cable de corriente y pulsa el botón de encendido unos segundos para descargar los condensadores.

---

## 1. Montaje físico

1. **RAM:** retira el módulo "Noname". Coloca los dos de 8 GB y los dos Hynix de 4 GB de forma que haya **un módulo de 8 en cada canal** (en la placa, las ranuras del mismo canal comparten color). Toca el chasis antes para descargar la electricidad estática.
2. **Kingston A400:** en la bahía del lector de DVD con el adaptador *caddy*.
3. **HDD de 2 TB:** en la bahía de 3,5", con su cable SATA y alimentación.
4. **Pasta térmica:** limpia la antigua con alcohol isopropílico y aplica una cantidad pequeña, como un grano de arroz, en el centro del procesador.

---

## 2. Validar antes de arrancar Proxmox

**1. Memtest86+** desde el USB de Ventoy: al menos una pasada completa sin errores. Si hay errores, prueba los módulos de uno en uno para encontrar el defectuoso. Esto también descarta la RAM como causa de las caídas de octubre.

**2. Arranca Proxmox y comprueba la memoria:**

```
free -h
```

La columna `total` debe rondar 23 Gi.

```
dmidecode -t memory | grep -E "Size|Locator|Speed" | grep -v Bank
```

`dmidecode -t memory` lee de la placa qué módulo hay en cada ranura. Confirma 8 + 8 + 4 + 4 GB.

**3. Identifica los discos nuevos:**

```
lsblk -o NAME,SIZE,MODEL,SERIAL,MOUNTPOINT
```

Compara con la captura del paso 0. El Kingston (240 GB) y el Samsung HD203WI (2 TB) aparecerán sin punto de montaje. **Apunta sus nombres** (por ejemplo `sdb` y `sdc`). En los comandos siguientes, `sdX` es el Kingston y `sdY` el HDD: sustitúyelos por los tuyos.

> ⚠️ `sda` es el disco del sistema. Si un comando de borrado apunta a `sda`, **para**.

**4. Prueba larga del HDD** (tiene 16 años; hay que saber si es de fiar):

```
smartctl -t long /dev/sdY
```

Lanza la prueba completa de superficie en segundo plano (varias horas). Puedes seguir con el SSD mientras tanto. Mira el resultado después con:

```
smartctl -a /dev/sdY | grep -E "result|Reallocated_Sector|Current_Pending|Offline_Uncorrectable"
```

`smartctl -a` muestra toda la información de salud; el filtro deja lo esencial. Lo que quieres ver: `Completed without error` y los tres contadores a **0**. Si hay sectores reasignados o pendientes, el disco sirve para una segunda copia, pero no como única copia.

---

## 3. Kingston como almacenamiento de VM (`vmdata`)

**1. Borra las firmas antiguas** (tiene las particiones del Windows clonado):

```
wipefs -a /dev/sdX && sgdisk --zap-all /dev/sdX
```

`wipefs -a` elimina las firmas de sistemas de archivos y `sgdisk --zap-all` borra las tablas de particiones. **Irreversible.**

**2. Crea el volumen LVM:**

```
pvcreate /dev/sdX
vgcreate vmdata /dev/sdX
lvcreate -l 95%FREE --thinpool data vmdata
```

- `pvcreate` marca el disco como volumen físico de LVM.
- `vgcreate vmdata` crea un grupo de volúmenes llamado `vmdata`.
- `lvcreate --thinpool` crea un *thin pool*: un espacio donde los discos virtuales solo ocupan lo que realmente usan. Se deja un 5 % libre para los metadatos.

**3. Añádelo a Proxmox:**

```
pvesm add lvmthin vmdata --vgname vmdata --thinpool data --content images,rootdir
```

`pvesm add` registra un almacenamiento; `images,rootdir` indica que acepta discos de VM y de contenedores. Comprueba:

```
pvesm status
```

Debe aparecer `vmdata` como `active`.

**4. Mueve el disco de docker01** (se puede hacer con la VM encendida):

```
qm disk move 103 scsi0 vmdata --delete 1
```

Mueve el disco `scsi0` de la VM 103 al almacenamiento nuevo; `--delete 1` borra la copia antigua al terminar. Tarda unos minutos. Comprueba:

```
qm config 103 | grep scsi0
```

Debe empezar por `vmdata:`.

**Por qué:** las VM dejan de escribir en el TR200 gastado, y si un disco muere no se lleva todo.

---

## 4. HDD de 2 TB para las copias

**1. Particiona y formatea:**

```
wipefs -a /dev/sdY && sgdisk --zap-all /dev/sdY
sgdisk -n 1:0:0 -t 1:8300 /dev/sdY
mkfs.ext4 -L backups /dev/sdY1
```

- `sgdisk -n 1:0:0` crea una partición que ocupa todo el disco; `-t 1:8300` la marca como Linux.
- `mkfs.ext4 -L backups` la formatea con la etiqueta `backups`.

**2. Móntalo siempre en el mismo sitio.** Los nombres `sdX` pueden cambiar entre arranques, así que se usa el identificador único (UUID):

```
mkdir -p /mnt/hdd2tb && blkid -s UUID -o value /dev/sdY1
```

Crea el punto de montaje y muestra el UUID. Cópialo y añade una línea a `/etc/fstab`:

```
nano /etc/fstab
```

```
UUID=PEGA-AQUI-EL-UUID  /mnt/hdd2tb  ext4  defaults,nofail  0  2
```

`nofail` es importante: si el disco falla, el servidor arranca igual en vez de quedarse esperando.

```
systemctl daemon-reload && mount -a && df -h /mnt/hdd2tb
```

`mount -a` monta todo lo de `fstab` (y te avisa si hay un error en la línea); `df -h` confirma ~1,8 T disponibles.

**3. Añádelo a Proxmox como destino de copias:**

```
pvesm add dir hdd-backup --path /mnt/hdd2tb --content backup --is_mountpoint 1
```

`--is_mountpoint 1` evita que, si el disco no está montado, Proxmox escriba las copias en el disco del sistema sin que te des cuenta.

**4. Cambia la tarea de copias:** Centro de datos → Respaldo → Editar → **Almacenamiento:** `hdd-backup`. En **Retención:** vuelve a `Keep Daily 7` y `Keep Weekly 2`. Ejecuta ahora y comprueba que la notificación muestra la ruta `/mnt/hdd2tb/dump/`.

**5. Actualiza la subida a Drive:**

```
nano /usr/local/bin/backup-to-drive.sh
```

Cambia `ORIGEN="/var/lib/vz/dump"` por `ORIGEN="/mnt/hdd2tb/dump"`. Ejecuta el script una vez y comprueba el aviso.

**6. Borra las copias viejas del disco del sistema** cuando confirmes que las nuevas están en el HDD y en Drive:

```
ls -lh /var/lib/vz/dump/ && rm /var/lib/vz/dump/vzdump-*
```

Lista primero (míralo) y después borra los archivos de copia antiguos.

---

## 5. Aprovechar la memoria nueva

```
qm set 103 --memory 6144 --balloon 3072
```

docker01 a 6 GB (mínimo 3). Aplica con `qm shutdown 103 && sleep 20 && qm start 103`.

Presupuesto orientativo con 24 GB: anfitrión 1,5 · contenedores 1,5 · docker01 6 · Wazuh 6–7 · DC 2–3 · resto libre para laboratorios que se encienden solo al usarlos.

---

## Verificación final de la fase B

- [ ] Memtest86+ sin errores, `free -h` ≈ 23 Gi
- [ ] SMART largo del HDD `Completed without error`
- [ ] `pvesm status` muestra `vmdata` y `hdd-backup` activos
- [ ] Disco de docker01 en `vmdata`
- [ ] Copia diaria en el HDD con retención 7 + 2, y subida a Drive funcionando desde la ruta nueva
- [ ] Temperatura en reposo igual o mejor que antes (Homepage)
- [ ] Inventario (sección 4 y 5) y ROADMAP actualizados

**Regla 3-2-1 completa:** datos vivos en SSD + copia en el HDD + copia cifrada en Drive.
