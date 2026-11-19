# Runbook 07 — Active Directory en Proxmox (fase F)

> **Objetivo:** reconstruir el dominio `lab.internal` desde cero y en el orden correcto, con un cliente Windows y un servidor Linux unidos.
> **Requisito:** fase B (memoria). **Lección que guía todo:** nombre, IP y zona horaria **antes** de promover (incidencias 9, 10 y 11).

---

## 1. Preparar las ISO

En Proxmox → `local` → **ISO Images**, sube:
- Windows Server (evaluación, del centro de evaluación de Microsoft).
- **VirtIO drivers** (`virtio-win.iso`, del proyecto Fedora). Sin ellos, Windows no ve el disco ni la red.

---

## 2. Crear la VM `ws-dc01`

En **pve01** (ajusta el nombre de la ISO al tuyo):

```
qm create 110 --name ws-dc01 --ostype win11 --machine q35 --bios ovmf \
  --cores 2 --cpu host --memory 3072 --balloon 0 \
  --scsihw virtio-scsi-single --scsi0 local-lvm:60,discard=on,ssd=1,iothread=1 \
  --efidisk0 local-lvm:1,efitype=4m,pre-enrolled-keys=1 --tpmstate0 local-lvm:1,version=v2.0 \
  --net0 virtio,bridge=vmbr0 --agent 1 --onboot 1 \
  --ide2 local:iso/WINDOWS-SERVER.iso,media=cdrom --ide0 local:iso/virtio-win.iso,media=cdrom
```

- `--bios ovmf` + `efidisk0` + `tpmstate0`: arranque UEFI con TPM, como un servidor moderno.
- `--balloon 0`: un controlador de dominio necesita memoria fija.
- `scsi0 local-lvm:60`: disco de 60 GB en el SSD del sistema (`local-lvm`).
- Las dos unidades `ide`: el instalador y los controladores VirtIO.

Arráncala y abre la consola. Durante la instalación:
1. Edición **Standard (Desktop Experience)** o Server Core si quieres ahorrar memoria.
2. Cuando no aparezcan discos: **Cargar controlador** → unidad VirtIO → `vioscsi\2k25\amd64` (o la carpeta de tu versión).
3. Tras instalar: ejecuta `virtio-win-guest-tools.exe` desde la unidad VirtIO (red, *ballooning* y agente QEMU).

---

## 3. Lo primero: nombre, IP y hora

En PowerShell como administrador, **antes de instalar ningún rol**:

```powershell
Rename-Computer -NewName "ws-dc01" -Restart
```

Pone el nombre definitivo y reinicia. Después:

```powershell
Get-NetAdapter
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.1.241 -PrefixLength 24 -DefaultGateway 192.168.1.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1,1.1.1.1
Set-TimeZone -Id "Romance Standard Time"
```

- `Get-NetAdapter` muestra el nombre real del adaptador (úsalo en `-InterfaceAlias`).
- `New-NetIPAddress` fija la IP `.241`.
- El DNS apunta a sí mismo, porque el DC será el servidor DNS del dominio.
- `Romance Standard Time` es la zona horaria de Madrid.

**Comprueba** (regla 3, configuración efectiva):

```powershell
hostname; Get-NetIPConfiguration; Get-TimeZone
```

Instantánea en Proxmox: `qm snapshot 110 antes-de-promover`.

---

## 4. Promover a controlador de dominio

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
```

Instala el rol de Active Directory y sus herramientas.

```powershell
Install-ADDSForest -DomainName "lab.internal" -DomainNetbiosName "LAB" -InstallDns
```

Crea un bosque nuevo con el dominio `lab.internal` e instala el DNS. Te pedirá la contraseña del modo de restauración (DSRM): al gestor. Reinicia solo.

**Verifica:**

```powershell
Get-ADDomain | Select-Object DNSRoot, NetBIOSName, PDCEmulator
Resolve-DnsName lab.internal
dcdiag /q
```

`dcdiag /q` solo muestra errores: lo ideal es que no muestre nada.

**Reenviadores DNS** (para que el DC resuelva internet):

```powershell
Set-DnsServerForwarder -IPAddress 1.1.1.1, 1.0.0.1
```

---

## 5. Estructura: OU, grupos y cuentas (modelo AGDLP)

**AGDLP:** las cuentas (**A**) van en grupos globales (**G**) por función; esos grupos van en grupos locales de dominio (**DL**) por recurso; y los permisos (**P**) se dan a los grupos locales. Nunca permisos directos a personas.

```powershell
$base = "DC=lab,DC=internal"
New-ADOrganizationalUnit -Name "Bastion" -Path $base
"Usuarios","Administradores","Equipos","Servidores","Grupos" | ForEach-Object {
    New-ADOrganizationalUnit -Name $_ -Path "OU=Bastion,$base"
}
```

Crea una OU raíz propia y sus subunidades. No se usan los contenedores por defecto, porque no admiten directivas de grupo.

```powershell
New-ADGroup -Name "G_Soporte" -GroupScope Global -Path "OU=Grupos,OU=Bastion,$base"
New-ADGroup -Name "DL_Compartido_Lectura" -GroupScope DomainLocal -Path "OU=Grupos,OU=Bastion,$base"
Add-ADGroupMember -Identity "DL_Compartido_Lectura" -Members "G_Soporte"
```

**Doble cuenta** (uso diario y administración, separadas):

```powershell
New-ADUser -Name "Martin De Prada" -SamAccountName "mdeprada" -Path "OU=Usuarios,OU=Bastion,$base" -AccountPassword (Read-Host -AsSecureString "Contraseña") -Enabled $true
New-ADUser -Name "Martin De Prada (adm)" -SamAccountName "adm-mdeprada" -Path "OU=Administradores,OU=Bastion,$base" -AccountPassword (Read-Host -AsSecureString "Contraseña") -Enabled $true
Add-ADGroupMember -Identity "Domain Admins" -Members "adm-mdeprada"
```

`Read-Host -AsSecureString` pide la contraseña sin mostrarla ni dejarla en el historial.

---

## 6. Integrar con Pi-hole

Para que los dispositivos de la tailnet resuelvan `lab.internal`: Pi-hole → **Settings → DNS → Conditional forwarding** → red `192.168.1.0/24`, servidor `192.168.1.241`, dominio `lab.internal`.

Comprueba desde tu móvil o PC con Tailscale:

```
nslookup ws-dc01.lab.internal 192.168.1.244
```

---

## 7. Cliente Windows `ws-cli01`

VM con Windows 11 Enterprise (evaluación), 4 GB, **encendida solo cuando se use**. Tras instalar con los controladores VirtIO:

```powershell
Rename-Computer -NewName "ws-cli01" -Restart
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.1.241
Add-Computer -DomainName "lab.internal" -OUPath "OU=Equipos,OU=Bastion,DC=lab,DC=internal" -Credential LAB\adm-mdeprada -Restart
```

El DNS del cliente es el DC (así encuentra el dominio), y `Add-Computer` lo une directamente en la OU correcta. Comprueba con `gpresult /r` al iniciar sesión con `mdeprada`.

**Una GPO de prueba** (por ejemplo, bloqueo de pantalla a los 10 minutos) enlazada a `OU=Equipos`, y verificada con `gpresult /r` y `gpupdate /force`.

---

## 8. Linux en el dominio: `rocky-srv-01`

VM Rocky Linux (1,5 GB, IP `.247`, DNS apuntando al DC).

```
sudo dnf install -y realmd sssd oddjob oddjob-mkhomedir adcli samba-common-tools
sudo realm discover lab.internal
sudo realm join lab.internal -U adm-mdeprada
```

- `realm discover` comprueba que ve el dominio (si falla, es DNS: escalera de red).
- `realm join` une la máquina y configura SSSD para iniciar sesión con usuarios del dominio.

```
id mdeprada@lab.internal
sudo realm permit -g G_Soporte@lab.internal
```

`id` confirma que el servidor reconoce al usuario del dominio; `realm permit -g` limita el acceso a un grupo concreto (mínimo privilegio).

---

## 9. Vigilar y proteger

- **Uptime Kuma:** monitor DNS a `192.168.1.241` (consulta `lab.internal`) y ping.
- **Wazuh:** instala el agente de Windows en el DC. Verás inicios de sesión fallidos, cambios en grupos privilegiados y creación de cuentas: las alertas clásicas de un SOC corporativo.
- **Copias:** entra sola en la tarea diaria (modo "Todo"). Prueba una restauración aislada, como con docker01.

---

## Verificación final de la fase F

- [ ] `dcdiag /q` sin errores
- [ ] Nombre, IP y zona horaria correctos **desde antes** de promover
- [ ] Estructura de OU, AGDLP y doble cuenta creada por PowerShell (guarda el script en GitHub, sin contraseñas)
- [ ] Cliente unido y GPO aplicada (`gpresult /r`)
- [ ] Rocky unido, acceso limitado a un grupo
- [ ] Resolución de `lab.internal` desde la tailnet
- [ ] Inventario y ROADMAP actualizados
