# 1. Infraestructura, Distribución y Red — Guía Replicable

**Rol:** 1. Infraestructura, Distro y Red (Benjamín)  
**Alcance:** Creación de la VM base, configuración de red en modo Bridge con IP fija, verificación de conectividad cliente-servidor, acceso SSH y snapshot base. *(No incluye instalación de Nginx, publicación del sitio, usuarios editores ni firewall, que corresponden a los roles 2 al 6.)*

---

## 1. Problema y Requerimientos

Se requiere levantar una máquina virtual Linux de tipo servidor que sirva como base estable, aislada y reproducible para desplegar la intranet estática ubicada en [`sitio/`](../sitio/).

### Requerimientos técnicos
- Sistema operativo GNU/Linux de servidor (sin entorno gráfico para reducir consumo y superficie de ataque).
- Conectividad de red en modo **Bridge** (`vmbr0`) para que la VM sea alcanzable directamente desde equipos cliente en la LAN.
- **IP fija** (`10.77.0.150/24`) para garantizar acceso predecible por parte del equipo y de los clientes.
- Acceso **SSH** habilitado y **snapshot base** creado antes de entregar la máquina al resto del equipo.

---

## 2. Análisis de Recursos Necesarios para Hostear `sitio/`

El contenido del repositorio en [`sitio/`](../sitio/) es un sitio **100% estático** (sin backend ni base de datos):
- `index.html`, `contacto.html`, `recursos.html` y `css/estilos.css`
- **Peso total en disco:** `24 KB`

| Recurso | Mínimo real para hostear `sitio/` (Ubuntu Server + Nginx) | Asignado a la VM (`intranet-so`) | Disponible en host `atlas` (Proxmox VE) |
| :--- | :--- | :--- | :--- |
| **vCPU** | 1 vCPU | **2 vCPU** (`host`) | 16 hilos (AMD Ryzen 7 7730U) |
| **RAM** | 512 MB (SO ~200 MB + Nginx ~15–30 MB) | **2 GB (2048 MB)** | 22 GiB total (~16 GiB disponibles) |
| **Disco** | 8–10 GB (SO + logs + respaldos) | **20 GB** (`local-lvm`, thin) | 776 GB libres en `local-lvm` |
| **Red** | 100 Mbps | **VirtIO Bridge (`vmbr0`)** | `10.77.0.0/24` (LAN local) |

> **Conclusión de dimensionamiento:** Para servir 24 KB de HTML/CSS estático, incluso 1 vCPU y 512 MB–1 GB de RAM serían suficientes. Se asignan **2 vCPU, 2 GB RAM y 20 GB de disco** para coincidir con la especificación del proyecto en [`README.md`](../README.md), dar holgura durante `apt update/upgrade`, compilaciones o pruebas de carga, y mantener margen de sobra en `atlas`.

---

## 3. Elección Justificada de Distribución: Ubuntu Server 24.04 LTS vs Debian 12

| Criterio | **Ubuntu Server 24.04 LTS (Elegida)** | **Debian 12 (Bookworm)** |
| :--- | :--- | :--- |
| **Soporte** | 5 años estándar (LTS hasta 2029) | ~3 años estándar + LTS comunitario |
| **Configuración de red** | **Netplan** (YAML declarativo en `/etc/netplan/`) | `ifupdown` (`/etc/network/interfaces`) o `systemd-networkd` |
| **Paquetes y kernel** | Más recientes y con soporte oficial de `cloud-init` | Muy estables, ciclo más conservador |
| **Documentación académica** | Amplísima y alineada con el `README.md` del equipo | Amplia, algo más manual en configuración inicial |
| **Consumo base** | Bajo (~200–250 MB RAM sin GUI) | Muy bajo (~150–200 MB RAM sin GUI) |

**Decisión:** Se elige **Ubuntu Server 24.04 LTS** porque combina estabilidad empresarial (5 años de soporte de seguridad), configuración declarativa de red con **Netplan**, excelente integración con `cloud-init` en Proxmox y coherencia directa con la arquitectura acordada por el equipo.

---

## 4. Topología de Red

```text
+--------------------------+        LAN (10.77.0.0/24)         +-------------------------------+
|  Equipo Cliente (Mac/PC) | <-------------------------------> |  Host Proxmox: atlas          |
|  Red local / SSH / Ping  |                                   |  IP Host: 10.77.0.10/24       |
+--------------------------+                                   |  Bridge:  vmbr0               |
                                                               |  Gateway: 10.77.0.1           |
                                                               +---------------+---------------+
                                                                               | Bridge (Capa 2)
                                                                               v
                                                               +-------------------------------+
                                                               |  VM 200: intranet-so          |
                                                               |  SO: Ubuntu Server 24.04 LTS  |
                                                               |  IP fija: 10.77.0.150/24      |
                                                               |  SSH: puerto 22               |
                                                               +-------------------------------+
```

- **Modo de red:** Bridge (`vmbr0`), exponiendo la VM como un equipo más dentro de la subred `10.77.0.0/24`.
- **IP fija:** `10.77.0.150/24`
- **Gateway:** `10.77.0.1`
- **DNS:** `8.8.8.8`, `1.1.1.1`

---

## 5. Despliegue Replicable en Proxmox VE (`atlas`)

El siguiente procedimiento permite recrear exactamente la misma VM desde cero en cualquier nodo Proxmox VE usando la imagen oficial Cloud de Ubuntu 24.04 LTS y `cloud-init`:

```bash
# Variables de configuración
VMID=200
VMNAME="intranet-so"
STORAGE="local-lvm"
IMG_DIR="/var/lib/vz/template/iso"
IMG_FILE="${IMG_DIR}/noble-server-cloudimg-amd64.img"
IMG_URL="https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img"

# 1. Descargar la imagen oficial de Ubuntu Server 24.04 LTS (Cloud Image)
mkdir -p "$IMG_DIR"
wget -O "$IMG_FILE" "$IMG_URL"

# 2. Crear la VM base (2 vCPU, 2 GB RAM, red Bridge vmbr0)
qm create "$VMID" \
  --name "$VMNAME" \
  --ostype l26 \
  --memory 2048 \
  --cores 2 \
  --cpu host \
  --net0 virtio,bridge=vmbr0 \
  --scsihw virtio-scsi-single \
  --serial0 socket \
  --vga serial0 \
  --agent enabled=1

# 3. Importar disco, adjuntar y expandir a 20 GB
qm importdisk "$VMID" "$IMG_FILE" "$STORAGE"
qm set "$VMID" --scsi0 "${STORAGE}:vm-${VMID}-disk-0,discard=on,ssd=1"
qm resize "$VMID" scsi0 20G

# 4. Configurar unidad cloud-init y orden de arranque
qm set "$VMID" --ide2 "${STORAGE}:cloudinit"
qm set "$VMID" --boot order=scsi0

# 5. Inyectar usuario, llave SSH e IP fija (genera /etc/netplan/50-cloud-init.yaml)
qm set "$VMID" \
  --ciuser ubuntu \
  --sshkeys /ruta/a/tu_llave.pub \
  --ipconfig0 ip=10.77.0.150/24,gw=10.77.0.1 \
  --nameserver "8.8.8.8 1.1.1.1"

# 6. Iniciar la VM
qm start "$VMID"
```

### Creación del Snapshot Base

Una vez verificada la conectividad y el acceso SSH, se guarda el estado limpio para poder compartir o restaurar la VM en cualquier momento:

```bash
qm snapshot 200 base-inicial --description "Ubuntu Server 24.04 LTS base con IP fija 10.77.0.150 y SSH"
qm listsnapshot 200
```

---

## 6. Verificación y Guion de Demo en Vivo (~2 min)

1. **Acceso SSH desde el equipo cliente hacia la VM:**
   ```bash
   ssh ubuntu@10.77.0.150
   ```
2. **Mostrar IP fija y tabla de rutas dentro de la VM:**
   ```bash
   ip -br a
   ip route
   sudo cat /etc/netplan/50-cloud-init.yaml
   ```
3. **Probar conectividad bidireccional (cliente ↔ servidor):**
   ```bash
   # Desde el equipo cliente hacia la VM
   ping -c 4 10.77.0.150

   # Desde la VM hacia el gateway
   ping -c 4 10.77.0.1
   ```
4. **Mostrar el snapshot base disponible en Proxmox (`atlas`):**
   ```bash
   qm listsnapshot 200
   ```
