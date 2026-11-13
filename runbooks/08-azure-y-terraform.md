# Runbook 08 — Azure bajo demanda y Terraform (fase G)

> **Objetivo:** aprender Azure y demostrar que tu infraestructura es portable: la misma web y el mismo rol de Ansible, en otra nube, creados con código.
> **Cuándo:** dos semanas antes de que caduque Google Cloud (tienes el recordatorio en el calendario).
> **Regla de esta fase:** en Azure todo se **enciende, se usa y se apaga**. 100 $ deben durar 12 meses.

---

## 1. Activar Azure for Students

1. Entra en la página de Azure for Students con tu correo de **Universae** (verificación académica). No pide tarjeta.
2. Recibes **100 $ para 12 meses** y servicios gratuitos, entre ellos 750 horas al mes de una VM B1s.
3. Instala la herramienta de línea de comandos en tu PC:

```
winget install -e --id Microsoft.AzureCLI
az login
az account show --output table
```

`az login` abre el navegador para iniciar sesión; `az account show` confirma que estás en la suscripción de estudiantes.

**Presupuesto, lo primero:** portal de Azure → **Cost Management → Budgets → Add**: 10 $ al mes con avisos al 50, 90 y 100 %.

---

## 2. Terraform: infraestructura como código

**Qué es:** escribes en archivos de texto qué infraestructura quieres (una VM, una red, una regla de cortafuegos) y Terraform la crea, la modifica o la destruye. El reparto con Ansible:

| Herramienta | Qué hace | Ejemplo |
|---|---|---|
| **Terraform** | Crea la infraestructura | "Una VM de 1 vCPU en Europa con los puertos 80 y 443 abiertos" |
| **Ansible** | Configura lo que hay dentro | "Usuario, SSH endurecido, Docker y el sitio web" |

```
winget install -e --id Hashicorp.Terraform
terraform version
```

En tu repositorio, crea `terraform/azure/main.tf`:

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {}
}

variable "ssh_public_key" {
  description = "Clave pública SSH para el usuario administrador"
  type        = string
}

resource "azurerm_resource_group" "bastion" {
  name     = "rg-bastion"
  location = "spaincentral"
}

resource "azurerm_virtual_network" "red" {
  name                = "vnet-bastion"
  address_space       = ["10.20.0.0/16"]
  location            = azurerm_resource_group.bastion.location
  resource_group_name = azurerm_resource_group.bastion.name
}

resource "azurerm_subnet" "subred" {
  name                 = "snet-web"
  resource_group_name  = azurerm_resource_group.bastion.name
  virtual_network_name = azurerm_virtual_network.red.name
  address_prefixes     = ["10.20.1.0/24"]
}

resource "azurerm_public_ip" "ip" {
  name                = "pip-bastion-web"
  location            = azurerm_resource_group.bastion.location
  resource_group_name = azurerm_resource_group.bastion.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_network_security_group" "nsg" {
  name                = "nsg-web"
  location            = azurerm_resource_group.bastion.location
  resource_group_name = azurerm_resource_group.bastion.name

  security_rule {
    name                       = "permitir-web"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_ranges    = ["80", "443"]
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

resource "azurerm_network_interface" "nic" {
  name                = "nic-bastion-web"
  location            = azurerm_resource_group.bastion.location
  resource_group_name = azurerm_resource_group.bastion.name

  ip_configuration {
    name                          = "interna"
    subnet_id                     = azurerm_subnet.subred.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.ip.id
  }
}

resource "azurerm_network_interface_security_group_association" "asociacion" {
  network_interface_id      = azurerm_network_interface.nic.id
  network_security_group_id = azurerm_network_security_group.nsg.id
}

resource "azurerm_linux_virtual_machine" "web" {
  name                  = "bastion-web-az"
  resource_group_name   = azurerm_resource_group.bastion.name
  location              = azurerm_resource_group.bastion.location
  size                  = "Standard_B1s"
  admin_username        = "martin"
  network_interface_ids = [azurerm_network_interface.nic.id]

  admin_ssh_key {
    username   = "martin"
    public_key = var.ssh_public_key
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Debian"
    offer     = "debian-13"
    sku       = "13-gen2"
    version   = "latest"
  }
}

output "ip_publica" {
  value = azurerm_public_ip.ip.ip_address
}
```

Cómo leerlo:
- Cada bloque `resource` es una pieza: grupo de recursos (la "carpeta" que lo contiene todo), red, subred, IP pública, cortafuegos (`nsg`, solo 80 y 443), tarjeta de red y la VM.
- Las referencias como `azurerm_resource_group.bastion.name` enlazan unas piezas con otras; Terraform deduce el orden de creación.
- `variable` evita escribir tu clave en el archivo; `output` muestra la IP al terminar.
- No se abre el puerto 22: se administra por Tailscale, igual que en Google Cloud. Para el primer acceso (instalar Tailscale), usa temporalmente la consola serie de Azure o `az vm run-command`.

> Los nombres de la imagen (`offer`, `sku`) y la región cambian con el tiempo. Compruébalos con `az vm image list --publisher Debian --all --output table` y `az account list-locations --output table`. La suscripción de estudiantes puede limitar las regiones disponibles.

**El ciclo de Terraform:**

```
cd terraform/azure
terraform init
```

Descarga el proveedor de Azure.

```
terraform plan -var "ssh_public_key=$(Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub)"
```

Muestra **qué va a crear** sin tocar nada. Léelo siempre: es tu red de seguridad.

```
terraform apply -var "ssh_public_key=$(Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub)"
```

Crea todo tras escribir `yes`. Al final imprime la IP pública.

**Importante para GitHub:** el `.gitignore` ya excluye `*.tfstate` y `.terraform/`. El archivo de estado contiene detalles de tu infraestructura y no se sube.

---

## 3. La prueba de portabilidad

Este es el momento que justifica todo el diseño:

1. Une la VM a Tailscale con `tag:nube` (runbook 05, paso 3).
2. Añádela al inventario de Ansible y aplica el rol `bootstrap`.
3. Copia `compose/web/` y levanta Caddy con un subdominio de prueba (`az.tudominio.com`).
4. Comprueba que la web carga con HTTPS.

**Apunta el tiempo que tardas.** "Desplegué el mismo servicio en una segunda nube en 20 minutos con Terraform y Ansible" es una frase de entrevista con una prueba detrás (el historial de *commits*).

Escribe después el equivalente para Google Cloud en `terraform/gcp/main.tf` (proveedor `google`, recursos `google_compute_instance` y `google_compute_firewall`): comparar los dos archivos enseña qué cambia entre nubes y qué no.

---

## 4. Apagar para no gastar

```
az vm deallocate --resource-group rg-bastion --name bastion-web-az
```

`deallocate` apaga la VM **y deja de facturar el cómputo** (apagar desde dentro del sistema no basta). El disco y la IP estática sí siguen costando un poco.

Cuando termines un laboratorio:

```
terraform destroy -var "ssh_public_key=x"
```

Borra todo lo que creó Terraform. Volver a crearlo es un `terraform apply`.

```
az resource list --output table
```

Comprueba que no queda nada olvidado.

---

## 5. Laboratorios para los 12 meses (uno cada vez)

| Laboratorio | Qué aprendes | Coste orientativo |
|---|---|---|
| **Entra ID** (el Active Directory de la nube) | Usuarios, grupos, acceso condicional, MFA. Es gratis en su nivel básico | 0 $ |
| **Entra Connect**: sincronizar tu `lab.internal` de casa con Entra ID | Identidad híbrida, lo que usan las multinacionales | VM pequeña unas horas |
| **Windows Server en Azure** unido al dominio | Administración remota, Azure Bastion | Solo encendida al practicar |
| **Microsoft Sentinel** con los registros de tu honeypot | SIEM en la nube, comparado con tu Wazuh | Tiene nivel gratuito limitado; vigila el consumo |
| **AKS** (Kubernetes) | Contenedores a escala | Caro: crear, practicar una tarde y destruir |

Encaja con certificaciones: **AZ-900** (fundamentos) y después **AZ-104** (administrador) o **SC-900** (seguridad).

---

## Verificación final de la fase G

- [ ] Presupuesto de 10 $/mes con alertas
- [ ] `terraform plan` limpio y `apply` crea la VM
- [ ] La web carga en Azure con el mismo rol de Ansible y el mismo Compose
- [ ] `terraform destroy` deja `az resource list` vacío
- [ ] `terraform/azure/` en GitHub, sin estado ni claves
- [ ] ROADMAP actualizado
