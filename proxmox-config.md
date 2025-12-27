# Configuración Proxmox VE - geekcon

> **Última actualización:** 26 Diciembre 2025  
> **IP:** 192.168.50.8  
> **Usuario:** root

---

# 🖥️ NODO PROXMOX (Host)

Todo en esta sección está configurado directamente en el nodo Proxmox.

---

## Sistema

| Componente | Valor |
|------------|-------|
| **Hostname** | geekcon |
| **Proxmox VE** | 9.1.0 |
| **Kernel** | 6.17.4-1-pve |
| **CPU** | AMD Ryzen AI 9 HX 370 w/ Radeon 890M |
| **Cores/Threads** | 12 cores / 24 threads |
| **RAM** | 30 GB |
| **GPU** | AMD Radeon 890M (integrada) |
| **NPU** | AMD XDNA (Ryzen AI) |

---

## � Repositorios APT

### Configuración para uso sin suscripción

Por defecto, Proxmox viene con el repositorio enterprise (de pago). Hay que deshabilitarlo y habilitar el repositorio gratuito.

```bash
# 1. Deshabilitar repositorio enterprise (renombrar)
mv /etc/apt/sources.list.d/pve-enterprise.sources /etc/apt/sources.list.d/pve-enterprise.sources.disabled

# 2. Crear repositorio no-subscription (gratuito)
cat > /etc/apt/sources.list.d/pve-no-subscription.sources << 'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF

# 3. Configurar Ceph sin suscripción (opcional, si usas Ceph)
cat > /etc/apt/sources.list.d/ceph.sources << 'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/ceph-squid
Suites: trixie
Components: no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF

# 4. Actualizar
apt update && apt upgrade -y
```

### Estado actual de los repositorios

| Archivo | Estado | Descripción |
|---------|--------|-------------|
| `pve-enterprise.sources.disabled` | ❌ Deshabilitado | Repo de pago |
| `pve-no-subscription.sources` | ✅ Habilitado | Repo gratuito Proxmox |
| `ceph.sources` | ✅ Habilitado | Ceph Squid (no-subscription) |
| `debian.sources` | ✅ Habilitado | Debian Trixie base |

---

## �💾 Almacenamiento

### Discos físicos

| Disco | Tipo | Tamaño | Uso |
|-------|------|--------|-----|
| **nvme1n1** | NVMe | 1.82 TB | Sistema + VMs (LVM) |
| **nvme0n1** | NVMe | 1.9 TB | Backups (`/mnt/backups`) |

### 📊 Esquema del disco del sistema (nvme1n1 - 1.82 TB)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DISCO FÍSICO: nvme1n1 (1.82 TB)                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────┐  ┌──────────┐  ┌────────────────────────────────────────────┐ │
│  │ nvme1n1p1│  │ nvme1n1p2│  │              nvme1n1p3                     │ │
│  │  1 MB    │  │  1 GB    │  │              1.82 TB                       │ │
│  │  (BIOS)  │  │ /boot/efi│  │            LVM2_member                     │ │
│  └──────────┘  └──────────┘  └────────────────────────────────────────────┘ │
│                                           │                                 │
└───────────────────────────────────────────┼─────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    VOLUME GROUP: pve (1.82 TB total)                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────────────────────────┐ │
│  │  pve-root  │  │  pve-swap  │  │           pve-data (THIN POOL)         │ │
│  │   96 GB    │  │   8 GB     │  │              1.71 TB                   │ │
│  │   ext4     │  │   swap     │  │                                        │ │
│  │            │  │            │  │  ┌──────────────────────────────────┐  │ │
│  │  Usado:    │  │            │  │  │       Espacio THIN POOL          │  │ │
│  │   5 GB     │  │            │  │  │                                  │  │ │
│  │  (6%)      │  │            │  │  │  Total:     1.71 TB              │  │ │
│  │            │  │            │  │  │  Usado:     21 GB (1.24%)        │  │ │
│  │  Libre:    │  │            │  │  │  Libre:    ~1.69 TB              │  │ │
│  │   85 GB    │  │            │  │  │                                  │  │ │
│  └────────────┘  └────────────┘  │  │  ┌────────────────────────────┐  │  │ │
│       │               │          │  │  │   vm-100-disk-0            │  │  │ │
│       ▼               ▼          │  │  │   (docker-commander)       │  │  │ │
│      /              [SWAP]       │  │  │                            │  │  │ │
│                                  │  │  │   Tamaño virtual: 32 GB    │  │  │ │
│                                  │  │  │   Usado real:    ~21 GB    │  │  │ │
│                                  │  │  │   (66% del virtual)        │  │  │ │
│                                  │  │  └────────────────────────────┘  │  │ │
│                                  │  │                                  │  │ │
│                                  │  │  🆓 Espacio para nuevas VMs/LXC  │  │ │
│                                  │  │     ~1.69 TB disponibles         │  │ │
│                                  │  └──────────────────────────────────┘  │ │
│                                  └────────────────────────────────────────┘ │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ 🔹 Espacio libre en VG (sin asignar): 16.24 GB                          ││
│  │    (Disponible para expandir pve-root, pve-swap o pve-data)             ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
```

### 💡 Thin Provisioning

| Concepto | Explicación |
|----------|-------------|
| **Thin Pool** | Es un "depósito" de espacio del que se asigna dinámicamente |
| **Tamaño virtual** | Lo que "ve" la VM/LXC (ej: 32 GB para docker-commander) |
| **Uso real** | Solo se consume espacio cuando se escriben datos realmente |

### 📁 Almacenamiento en Proxmox

| Storage | Tipo | Contenido | Tamaño |
|---------|------|-----------|--------|
| `local` | dir | ISOs, templates, snippets | 94 GB |
| `local-lvm` | lvmthin | Discos de VMs/LXC | 1.71 TB |
| `backup-storage` | dir | Backups, ISOs, templates | 1.9 TB |

---

### 💿 Disco de Backups (nvme0n1 - 1.9 TB)

Este disco **NO se formatea** en una reinstalación. Contiene los backups y datos persistentes.

| Propiedad | Valor |
|-----------|-------|
| **Disco** | nvme0n1 |
| **Tamaño** | 1.9 TB |
| **Formato** | ext4 |
| **Montaje** | `/mnt/backups` |
| **UUID actual** | `ddee81bf-1ac8-4ea3-b1e4-d28c2f74158d` |
| **Storage Proxmox** | `backup-storage` |

#### Configuración tras reinstalación

```bash
# 1. Crear punto de montaje
mkdir -p /mnt/backups

# 2. Obtener UUID del disco (puede haber cambiado el nombre del dispositivo)
# Buscar el disco de ~1.9TB con formato ext4
lsblk -o NAME,SIZE,FSTYPE,UUID
# O específicamente:
blkid | grep ext4

# 3. Añadir a fstab usando el UUID detectado
# Reemplazar XXXXXXXX con el UUID real del disco
UUID_BACKUP=$(blkid -s UUID -o value /dev/nvme0n1)
echo "UUID=$UUID_BACKUP /mnt/backups ext4 defaults 0 2" >> /etc/fstab

# 4. Montar
mount -a

# 5. Verificar
df -h /mnt/backups
ls -la /mnt/backups/
```

#### Registrar como storage en Proxmox

```bash
# Añadir storage (si no existe)
pvesm add dir backup-storage --path /mnt/backups --content vztmpl,backup,snippets,iso

# Verificar
pvesm status
```

#### Contenido esperado

```
/mnt/backups/
├── dump/        # Backups de VMs/LXC (vzdump)
├── images/      # (vacío normalmente)
├── snippets/    # Scripts y configs
├── template/    # Templates de contenedores
│   ├── cache/
│   └── iso/     # ISOs
└── lost+found/
```

---

## 🌐 Red

**Archivo:** `/etc/network/interfaces`

```
vmbr0 (Bridge Principal)
├── IP: 192.168.50.8/24
├── Gateway: 192.168.50.1
├── Puerto físico: nic0
└── Wake-on-LAN: Habilitado (ethtool -s nic0 wol g)
```

**Interfaces adicionales:** nic1, nic2 (manual, sin configurar)

---

## ⚡ Tuned - Optimización CPU

**Paquete:** tuned  
**Perfil activo:** `proxmox-ryzen`

### Instalación

```bash
# 1. Instalar tuned
apt update && apt install tuned -y

# 2. Crear directorio del perfil
mkdir -p /etc/tuned/profiles/proxmox-ryzen

# 3. Crear archivo de configuración
cat > /etc/tuned/profiles/proxmox-ryzen/tuned.conf << 'EOF'
[main]
include=virtual-host

[cpu]
# CRÍTICO: Para amd_pstate_epp, 'powersave' habilita el modo dinámico.
# Si se deja en 'performance', el EPP se bloquea y no cambia.
governor=powersave
energy_performance_preference=balance_performance
EOF

# 4. Activar el perfil
tuned-adm profile proxmox-ryzen

# 5. Verificar
tuned-adm active
```

| Parámetro | Valor | Propósito |
|-----------|-------|-----------|
| **include** | `virtual-host` | Hereda config optimizada para hipervisores |
| **governor** | `powersave` | Con `amd_pstate_epp`, permite escalado dinámico |
| **energy_performance_preference** | `balance_performance` | Equilibrio rendimiento/eficiencia |

---

## 🌡️ Monitorización Térmica

### Instalación

```bash
# 1. Instalar lm-sensors
apt install lm-sensors -y

# 2. Crear el script de vigilancia
cat > /usr/local/bin/temp-watchdog.sh << 'EOF'
#!/bin/bash
# CONFIGURACION
UMBRAL_WARN=85
UMBRAL_CRIT=95
SENSOR_CMD="sensors"

# Obtener temperatura actual de Tctl (Ryzen)
# Extrae solo el numero flotante
TEMP=$(sensors -u k10temp-pci-00c3 | grep "temp1_input" | awk '{print $2}')
TEMP_INT=${TEMP%.*}

# Logica de Alerta
if [ "$TEMP_INT" -ge "$UMBRAL_CRIT" ]; then
    msg="CRITICAL: CPU Temperature at ${TEMP}°C. Risk of thermal shutdown."
    echo "$msg" | logger -t temp-watchdog -p user.emerg
    # Descomentar si tienes email configurado en Proxmox
    # echo "$msg" | mail -s "PVE THERMAL CRITICAL" root
elif [ "$TEMP_INT" -ge "$UMBRAL_WARN" ]; then
    msg="WARNING: CPU Temperature high (${TEMP}°C)."
    echo "$msg" | logger -t temp-watchdog -p user.alert
fi
EOF

# 3. Hacer ejecutable
chmod +x /usr/local/bin/temp-watchdog.sh

# 4. Crear tarea cron (ejecutar cada minuto)
cat > /etc/cron.d/temp-watchdog << 'EOF'
* * * * * root /usr/local/bin/temp-watchdog.sh
EOF

# 5. Verificar que funciona
/usr/local/bin/temp-watchdog.sh
sensors | grep Tctl
```

### Configuración

| Umbral | Temperatura | Acción |
|--------|-------------|--------|
| ⚠️ **WARNING** | ≥ 85°C | Log a syslog (user.alert) |
| 🔴 **CRITICAL** | ≥ 95°C | Log a syslog (user.emerg) |

**Ver alertas:** `journalctl -t temp-watchdog`

---

## 📊 Script sysinfo (estado del sistema)

Script para ver rápidamente el estado del nodo con temperaturas, recursos y VMs.

### Instalación

```bash
cat > /usr/local/bin/sysinfo << 'EOF'
#!/bin/bash
GREEN='\033[32m'; YELLOW='\033[33m'; RED='\033[31m'; CYAN='\033[36m'; BOLD='\033[1m'; NC='\033[0m'

color_temp() {
  t=${1%.*}
  [[ $t -ge 80 ]] && echo -e "${RED}$1°C${NC}" && return
  [[ $t -ge 60 ]] && echo -e "${YELLOW}$1°C${NC}" && return
  echo -e "${GREEN}$1°C${NC}"
}

CPU=$(sensors k10temp-pci-00c3 2>/dev/null | awk '/Tctl/{gsub(/[+°C]/,"",$2);print $2}')
GPU=$(sensors amdgpu-pci-c700 2>/dev/null | awk '/edge/{gsub(/[+°C]/,"",$2);print $2}')
PWR=$(sensors amdgpu-pci-c700 2>/dev/null | awk '/PPT/{print $2}')
NV1=$(sensors nvme-pci-c100 2>/dev/null | awk '/Composite/{gsub(/[+°C]/,"",$2);print $2}')
NV2=$(sensors nvme-pci-c600 2>/dev/null | awk '/Composite/{gsub(/[+°C]/,"",$2);print $2}')
RAM1=$(sensors spd5118-i2c-2-50 2>/dev/null | awk '/temp1/{gsub(/[+°C]/,"",$2);print $2;exit}')
RAM2=$(sensors spd5118-i2c-2-51 2>/dev/null | awk '/temp1/{gsub(/[+°C]/,"",$2);print $2;exit}')
WIFI=$(sensors mt7925_phy0-pci-c300 2>/dev/null | awk '/temp1/{gsub(/[+°C]/,"",$2);print $2}')
CPU_PWR=$(turbostat --Summary --quiet --show PkgWatt sleep 1 2>&1 | tail -1)
RAM_U=$(free -h | awk '/Mem/{print $3}'); RAM_T=$(free -h | awk '/Mem/{print $2}')
DISK=$(df -h / | awk 'NR==2{print $5}')
VMS=$(qm list 2>/dev/null | grep -c running); LXC=$(pct list 2>/dev/null | grep -c running)
BIOS=$(dmidecode -s bios-version 2>/dev/null)
BIOS_DATE=$(dmidecode -s bios-release-date 2>/dev/null)
EC=$(cat /sys/class/dmi/id/ec_firmware_release 2>/dev/null || echo "N/A")

echo -e "${BOLD}${CYAN}═══════════════════════════════════════════════${NC}"
echo -e "${BOLD}${CYAN}  🖥️  PROXMOX: geekcon${NC}"
echo -e "${BOLD}${CYAN}═══════════════════════════════════════════════${NC}"
echo -e "${CYAN}  🔧  Firmware${NC}"
echo -e "      BIOS:           ${BIOS} (${BIOS_DATE})"
echo -e "      EC:             ${EC}"
echo -e "${CYAN}  🌡️  Temperaturas${NC}"
echo -e "      CPU (Ryzen):    $(color_temp $CPU)"
echo -e "      GPU (890M):     $(color_temp $GPU) (${PWR}W)"
echo -e "      NVMe Sistema:   $(color_temp $NV1)"
echo -e "      NVMe Backups:   $(color_temp $NV2)"
echo -e "      RAM:            $(color_temp $RAM1) / $(color_temp $RAM2)"
echo -e "      WiFi:           $(color_temp $WIFI)"
echo -e "${CYAN}  ⚡  Consumo${NC}"
echo -e "      CPU:            ${CPU_PWR}W"
echo -e "      GPU:            ${PWR}W"
echo -e "${CYAN}  📊  Recursos${NC}"
echo -e "      RAM:            $RAM_U / $RAM_T"
echo -e "      Disco /:        $DISK"
echo -e "${CYAN}  🖼️  Virtualización${NC}"
echo -e "      VMs:            $VMS corriendo"
echo -e "      LXC:            $LXC corriendo"
echo -e "${BOLD}${CYAN}═══════════════════════════════════════════════${NC}"
EOF

chmod +x /usr/local/bin/sysinfo
```

### Uso

```bash
sysinfo
```

---

## � Wake-on-LAN

Permite encender el servidor remotamente desde la red.

### Estado actual

```
Supports Wake-on: pumbg
Wake-on: g (Magic Packet)
```

### Instalación

El WoL se configura en `/etc/network/interfaces` como post-up en la interfaz física:

```bash
# Ya incluido en la configuración de red:
auto nic0
iface nic0 inet manual
        post-up /usr/sbin/ethtool -s nic0 wol g
```

### Verificar

```bash
ethtool nic0 | grep Wake
# Debe mostrar: Wake-on: g
```

### Encender remotamente

Desde otro equipo en la red:

```bash
# Linux
wakeonlan BC:24:11:XX:XX:XX

# O con etherwake
etherwake -i eth0 BC:24:11:XX:XX:XX
```

> **Nota:** También debe estar habilitado en la BIOS del mini PC.

---

## 🚫 Quitar aviso de suscripción (Subscription Nag)

Script que elimina el popup de suscripción de la interfaz web y móvil de Proxmox.

### Instalación

```bash
# 1. Crear el script
cat > /usr/local/bin/pve-remove-nag.sh << 'EOF'
#!/bin/sh
WEB_JS=/usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
if [ -s "$WEB_JS" ] && ! grep -q NoMoreNagging "$WEB_JS"; then
    echo "Patching Web UI nag..."
    sed -i -e "/data\.status/ s/!//" -e "/data\.status/ s/active/NoMoreNagging/" "$WEB_JS"
fi

MOBILE_TPL=/usr/share/pve-yew-mobile-gui/index.html.tpl
MARKER="<!-- MANAGED BLOCK FOR MOBILE NAG -->"
if [ -f "$MOBILE_TPL" ] && ! grep -q "$MARKER" "$MOBILE_TPL"; then
    echo "Patching Mobile UI nag..."
    printf "%s\n" \
      "$MARKER" \
      "<script>" \
      "  function removeSubscriptionElements() {" \
      "    const dialogs = document.querySelectorAll('dialog.pwt-outer-dialog');" \
      "    dialogs.forEach(dialog => {" \
      "      const text = (dialog.textContent || '').toLowerCase();" \
      "      if (text.includes('subscription')) {" \
      "        dialog.remove();" \
      "      }" \
      "    });" \
      "    const cards = document.querySelectorAll('.pwt-card.pwt-p-2.pwt-d-flex.pwt-interactive.pwt-justify-content-center');" \
      "    cards.forEach(card => {" \
      "      const text = (card.textContent || '').toLowerCase();" \
      "      const hasButton = card.querySelector('button');" \
      "      if (!hasButton && text.includes('subscription')) {" \
      "        card.remove();" \
      "      }" \
      "    });" \
      "  }" \
      "  const observer = new MutationObserver(removeSubscriptionElements);" \
      "  observer.observe(document.body, { childList: true, subtree: true });" \
      "  removeSubscriptionElements();" \
      "  setInterval(removeSubscriptionElements, 300);" \
      "  setTimeout(() => {observer.disconnect();}, 10000);" \
      "</script>" \
      "" >> "$MOBILE_TPL"
fi
EOF

# 2. Hacer ejecutable
chmod +x /usr/local/bin/pve-remove-nag.sh

# 3. Configurar para que se ejecute automáticamente tras cada apt upgrade
cat > /etc/apt/apt.conf.d/no-nag-script << 'EOF'
DPkg::Post-Invoke { "/usr/local/bin/pve-remove-nag.sh"; };
EOF

# 4. Ejecutar ahora
/usr/local/bin/pve-remove-nag.sh

# 5. Reiniciar servicio web
systemctl restart pveproxy
```

### Qué hace

| Componente | Acción |
|------------|--------|
| **Web UI** | Modifica `proxmoxlib.js` para saltar la comprobación de suscripción |
| **Mobile UI** | Inyecta JavaScript que elimina diálogos y cards de suscripción |
| **Auto-ejecución** | Se ejecuta automáticamente después de cada `apt upgrade` |

---

## 🖼️ Máquinas Virtuales

### VM 200: Windows 11 Pro

| Propiedad | Valor |
|-----------|-------|
| **VMID** | 200 |
| **Nombre** | windows11 |
| **OS** | Windows 11 Pro 25H2 |
| **CPU** | 8 cores (host) |
| **RAM** | 16 GB |
| **Disco** | 64 GB (VirtIO SCSI, SSD) |
| **Red** | VirtIO, bridge vmbr0 |
| **BIOS** | UEFI (OVMF) |
| **TPM** | 2.0 |
| **Display** | VirtIO (sin GPU passthrough) |
| **MAC** | BC:24:11:95:54:95 |

#### Acceso remoto

- **RDP:** Habilitado (puerto 3389)
- **Conexión:** `mstsc /v:[IP-Windows]` o Microsoft Remote Desktop en Mac

#### Recrear VM desde cero

```bash
# 1. Crear VM base
qm create 200 \
    --name windows11 \
    --memory 16384 \
    --cores 8 \
    --cpu host \
    --machine q35 \
    --bios ovmf \
    --net0 virtio,bridge=vmbr0 \
    --scsihw virtio-scsi-single \
    --scsi0 local-lvm:64,ssd=1,discard=on \
    --ostype win11 \
    --tpmstate0 local-lvm:1,version=v2.0 \
    --efidisk0 local-lvm:1 \
    --vga virtio

# 2. Conectar ISOs para instalación
qm set 200 --sata0 backup-storage:iso/es-es_windows_11.iso,media=cdrom
qm set 200 --sata1 backup-storage:iso/virtio-win.iso,media=cdrom
qm set 200 --boot order='sata0;scsi0'

# 3. Iniciar e instalar
qm start 200
# Durante instalación: cargar driver VirtIO desde D:\amd64\w11

# 4. Tras instalar, quitar ISOs
qm set 200 --delete sata0 --delete sata1
qm set 200 --boot order='scsi0'
```

#### Post-instalación

1. Instalar drivers VirtIO: `D:\virtio-win-guest-tools.exe`
2. Activar RDP: Configuración → Sistema → Escritorio remoto
3. Windows Update

---

# 📦 CONTENEDOR LXC: alpine-vaultwarden (CT 101)

Gestor de contraseñas compatible con Bitwarden, instalado via ProxMenux.

## Configuración

| Propiedad | Valor |
|-----------|-------|
| **VMID** | 101 |
| **OS** | Alpine Linux |
| **Hostname** | alpine-vaultwarden |
| **Cores** | 1 |
| **RAM** | 256 MB |
| **Swap** | 512 MB |
| **Disco** | 1 GB (local-lvm) |
| **Red** | DHCP via vmbr0 |
| **MAC** | BC:24:11:41:45:BE |
| **IP** | 192.168.50.148 |
| **Autostart** | Sí |
| **Features** | nesting=1, keyctl=1 |
| **Unprivileged** | Sí |

## Acceso

| Servicio | URL |
|----------|-----|
| **Vaultwarden Web** | `https://192.168.50.148:8000` |

## Certificado SSL (mkcert)

La app móvil de Bitwarden requiere HTTPS con un certificado válido. Para uso local/VPN, usamos **mkcert** para crear una CA (Autoridad Certificadora) local.

| Propiedad | Valor |
|-----------|-------|
| **Tipo** | Certificado local firmado por CA propia |
| **Válido para** | `192.168.50.148`, `vaultwarden.local`, `localhost` |
| **Expira** | Marzo 2028 |
| **Certificado** | `/etc/ssl/certs/vaultwarden-selfsigned.crt` |
| **Clave** | `/etc/ssl/private/vaultwarden-selfsigned.key` |

### Pasos realizados para configurar HTTPS

#### 1. Instalar mkcert en Mac

```bash
brew install mkcert
```

#### 2. Crear CA local e instalar en el sistema

```bash
mkcert -install
# Pide contraseña sudo para añadir la CA al llavero del sistema
```

Esto crea la CA en `~/Library/Application Support/mkcert/rootCA.pem`.

#### 3. Generar certificado para Vaultwarden

```bash
mkcert -cert-file /tmp/vaultwarden.crt -key-file /tmp/vaultwarden.key \
  192.168.50.148 vaultwarden.local localhost
```

#### 4. Copiar certificados al servidor Proxmox

```bash
scp /tmp/vaultwarden.crt /tmp/vaultwarden.key root@192.168.50.8:/tmp/
```

#### 5. Instalar certificados en el LXC 101 y reiniciar

```bash
ssh root@192.168.50.8 "pct push 101 /tmp/vaultwarden.crt /etc/ssl/certs/vaultwarden-selfsigned.crt && \
  pct push 101 /tmp/vaultwarden.key /etc/ssl/private/vaultwarden-selfsigned.key && \
  pct exec 101 -- rc-service vaultwarden restart"
```

---

### Instalar CA en dispositivos móviles

Para que la app de Bitwarden confíe en el certificado, hay que instalar la CA de mkcert en cada dispositivo.

**Obtener el archivo CA:**
```bash
# Ver ubicación
mkcert -CAROOT
# Copiar a ubicación accesible
cp "$(mkcert -CAROOT)/rootCA.pem" ~/Desktop/mkcert-CA.crt
```

#### iOS (iPhone/iPad)

1. **Enviar el archivo** `mkcert-CA.crt` al iPhone por AirDrop, email, o iCloud Drive
2. **Abrir el archivo** → Aparece "Perfil descargado"
3. **Instalar perfil:**
   - Ir a **Ajustes → General → VPN y gestión de dispositivos**
   - Tocar el perfil descargado → **Instalar**
   - Introducir código de desbloqueo si lo pide
4. **Activar confianza completa:**
   - Ir a **Ajustes → General → Información → Ajustes de certificados**
   - En "Activar confianza total para certificados raíz", activar el certificado de mkcert
5. **Configurar app Bitwarden:**
   - Abrir Bitwarden → Tocar engranaje (⚙️) antes de login
   - Seleccionar **Self-hosted**
   - URL: `https://192.168.50.148:8000`
   - Guardar y crear cuenta/login

#### Android

1. **Copiar el archivo** `mkcert-CA.crt` al dispositivo (cable USB, email, Drive, etc.)
2. **Instalar certificado:**
   - Ir a **Ajustes → Seguridad → Más ajustes de seguridad**
   - Tocar **Instalar desde almacenamiento del dispositivo**
   - Seleccionar el archivo `mkcert-CA.crt`
   - Dar un nombre (ej: "mkcert local")
   - Seleccionar "VPN y apps" como uso
3. **Configurar app Bitwarden:**
   - Abrir Bitwarden → Tocar engranaje (⚙️) antes de login
   - Seleccionar **Self-hosted**
   - URL: `https://192.168.50.148:8000`
   - Guardar y crear cuenta/login

> **Nota:** En algunos Android, la ruta puede variar. Buscar "Instalar certificado" en Ajustes.

---

### Regenerar certificado (si expira o cambia la IP)

```bash
# En Mac (requiere mkcert instalado)
mkcert -cert-file /tmp/vaultwarden.crt -key-file /tmp/vaultwarden.key \
  192.168.50.148 vaultwarden.local localhost

# Copiar al servidor y reiniciar
scp /tmp/vaultwarden.* root@192.168.50.8:/tmp/
ssh root@192.168.50.8 "pct push 101 /tmp/vaultwarden.crt /etc/ssl/certs/vaultwarden-selfsigned.crt && \
  pct push 101 /tmp/vaultwarden.key /etc/ssl/private/vaultwarden-selfsigned.key && \
  pct exec 101 -- rc-service vaultwarden restart"
```

## Prompt personalizado (Alpine sh)

Archivo: `/root/.profile`

```bash
export PS1="\033[1;32m\u\033[0m@\033[1;35mvaultwarden\033[0m:\033[1;34m\w\033[0m\$ "
```

Resultado: `root`(verde)`@vaultwarden`(magenta)`:/ruta`(azul)`$`

## Instalación (via ProxMenux)

Instalado automáticamente con:
```bash
menu → LXC → Vaultwarden
```

---

# 📦 CONTENEDOR LXC: docker-commander (CT 100)

Todo en esta sección está configurado en el contenedor LXC o es su definición desde Proxmox.

---

## Creación del contenedor

### Paso 1: Descargar template Debian 13 (Trixie)

```bash
# Actualizar lista de templates
pveam update

# Buscar última versión de Debian 13
pveam available | grep "debian-13-standard"
# Ejemplo de salida: debian-13-standard_13.1-2_amd64.tar.zst

# Descargar la versión que aparezca en backup-storage (persiste tras reinstalar)
pveam download backup-storage debian-13-standard_13.1-2_amd64.tar.zst
```

### Paso 2: Crear el contenedor

```bash
# Crear contenedor con configuración básica
# IMPORTANTE: La MAC está fija para que el router asigne siempre la misma IP
# Usar el template descargado (ajustar versión si es diferente)
pct create 100 backup-storage:vztmpl/debian-13-standard_13.1-2_amd64.tar.zst \
    --hostname docker-commander \
    --storage local-lvm \
    --rootfs local-lvm:32 \
    --memory 12288 \
    --swap 2048 \
    --cores 4 \
    --net0 name=eth0,bridge=vmbr0,hwaddr=BC:24:11:AF:E8:2E,ip=dhcp \
    --features nesting=1,keyctl=1 \
    --onboot 1 \
    --unprivileged 1
```

> **Nota:** MAC `BC:24:11:AF:E8:2E` → IP asignada por router: `192.168.50.67`

### Paso 3: Añadir configuración avanzada (GPU + CPU Pinning)

```bash
# Editar configuración del contenedor
cat >> /etc/pve/lxc/100.conf << 'EOF'

# --- OPTIMIZACION HARDWARE (Zen 5c Pinning) ---
# Tu CPU tiene 24 hilos (0 a 23).
# 0-7: P-Cores (Alto rendimiento - Reservados para el Host/VMs Gaming).
# 8-23: E-Cores (Eficiencia - Asignados a este contenedor).
lxc.cgroup2.cpuset.cpus: 8-23

# --- GPU & NPU PASSTHROUGH (AMD Strix Point) ---
# Mapeo verificado para Kernel 6.17 (25-Dic-2025)
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.cgroup2.devices.allow: c 234:0 rwm
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
lxc.mount.entry: /dev/kfd dev/kfd none bind,optional,create=file
EOF
```

### Paso 4: Iniciar y configurar Docker dentro del contenedor

```bash
# Iniciar contenedor
pct start 100

# Entrar al contenedor
pct enter 100

# Dentro del contenedor: instalar Docker
apt update && apt upgrade -y
apt install -y curl gnupg lsb-release ca-certificates

# Añadir repo Docker
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list

# Instalar Docker
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Verificar
docker --version
docker compose version

# Salir del contenedor
exit
```

---

## Configuración final del contenedor

**Archivo:** `/etc/pve/lxc/100.conf`

| Propiedad | Valor |
|-----------|-------|
| **VMID** | 100 |
| **OS** | Debian 12 |
| **Hostname** | docker-commander |
| **Cores** | 4 (pinned a E-Cores 8-23) |
| **RAM** | 12 GB |
| **Swap** | 2 GB |
| **Disco** | 32 GB (local-lvm) |
| **Red** | DHCP via vmbr0 |
| **Autostart** | Sí |
| **Features** | nesting=1, keyctl=1 |

---

## Prompt personalizado (Bash)

Archivo: `/root/.bashrc`

```bash
# Prompt con colores
export PS1="\[\033[1;32m\]\u\[\033[0m\]@\[\033[1;36m\]docker-commander\[\033[0m\]:\[\033[1;34m\]\w\[\033[0m\]\$ "

# Alias útiles
alias ll="ls -la --color=auto"
alias dc="docker compose"
alias dps="docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'"
```

Resultado: `root`(verde)`@docker-commander`(cyan)`:/ruta`(azul)`$`

## 💾 Backup y Restauración

### Backup existente

```
/mnt/backups/dump/vzdump-lxc-100-2025_12_25-21_24_47.tar.zst
```

### Crear backup manual

```bash
# Backup con snapshot (sin downtime)
vzdump 100 --storage backup-storage --mode snapshot --compress zstd

# Backup deteniendo el contenedor (más consistente)
vzdump 100 --storage backup-storage --mode stop --compress zstd
```

### Restaurar backup

#### Opción A: Interfaz web
1. **Datacenter** → **Storage** → `backup-storage`
2. Pestaña **Content** → Seleccionar backup
3. Clic derecho → **Restore**
4. Elegir VMID y storage destino

#### Opción B: Línea de comandos

```bash
# Ver backups disponibles
ls -la /mnt/backups/dump/

# Restaurar sobreescribiendo el contenedor actual
pct stop 100
pct restore 100 /mnt/backups/dump/vzdump-lxc-100-2025_12_25-21_24_47.tar.zst \
    --storage local-lvm
pct start 100

# O restaurar como nuevo contenedor
pct restore 101 /mnt/backups/dump/vzdump-lxc-100-2025_12_25-21_24_47.tar.zst \
    --storage local-lvm \
    --hostname docker-commander-restored
```

#### Opciones de restauración

| Opción | Uso |
|--------|-----|
| `--storage local-lvm` | Donde guardar el disco |
| `--hostname nuevo-nombre` | Cambiar hostname |
| `--force` | Sobreescribir si existe |
| `--unprivileged` | Mantener como unprivileged |

> **Nota:** Tras restaurar, verificar que la configuración de GPU passthrough y CPU pinning sigue correcta en `/etc/pve/lxc/100.conf`

---

## GPU/NPU Passthrough (AMD Strix Point)

| Dispositivo | Major:Minor | Descripción |
|-------------|-------------|-------------|
| `/dev/dri/card0` | 226:0 | GPU AMD Radeon 890M |
| `/dev/dri/renderD128` | 226:128 | Render node (aceleración) |
| `/dev/kfd` | 234:0 | ROCm/OpenCL |

### Verificar GPU dentro del contenedor

```bash
pct enter 100
ls -la /dev/dri/
# Debería mostrar: card0, renderD128
```

---

## 🐳 Stacks de Portainer

Los servicios Docker se gestionan desde **Portainer** (http://[IP-contenedor]:9000) usando Stacks.

### Paso 1: Instalar Portainer primero (único servicio con docker run)

```bash
pct enter 100

docker volume create portainer_data

docker run -d \
  --name portainer \
  --restart unless-stopped \
  -p 9000:9000 \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

### Paso 2: Crear stacks desde Portainer

Desde la web de Portainer → **Stacks** → **Add stack** → pegar el YAML.

---

### Stack: home-assistant

**Directorios a crear:**
```bash
mkdir -p /opt/homeassistant/config
```

```yaml
version: '3'

services:
  homeassistant:
    container_name: homeassistant
    image: "homeassistant/home-assistant:stable"
    volumes:
      - /opt/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone:ro
    restart: unless-stopped
    network_mode: host
```

---

### Stack: adguard

**Directorios a crear:**
```bash
mkdir -p /opt/adguard/work /opt/adguard/conf
```

```yaml
version: "3.8"

services:
  adguard:
    image: adguard/adguardhome:latest
    container_name: adguard
    restart: unless-stopped
    network_mode: host
    volumes:
      - /opt/adguard/work:/opt/adguardhome/work
      - /opt/adguard/conf:/opt/adguardhome/conf
```

### Script: Prewarm DNS (precalentar caché)

Mantiene los dominios más usados siempre en caché, refrescándolos cada 20 minutos.

**Lista de dominios:** `/root/scripts/domains.txt`

```
# CDNs
cdn.jsdelivr.net
cdnjs.cloudflare.com
fonts.googleapis.com
fonts.gstatic.com

# YouTube / Twitch
www.youtube.com
i.ytimg.com
www.twitch.tv
static.twitchcdn.net

# Google / GitHub
www.google.com
www.googleapis.com
github.com
raw.githubusercontent.com
```

**Script:** `/root/scripts/prewarm-dns.sh`

```bash
#!/bin/bash
DOMAINS_FILE="/root/scripts/domains.txt"
LOG="/var/log/prewarm-dns.log"

echo "$(date '+%H:%M:%S') Prewarm iniciado" >> "$LOG"
COUNT=0
while IFS= read -r domain; do
    [[ -z "$domain" || "$domain" == \#* ]] && continue
    dig +short +time=2 "$domain" @127.0.0.1 >/dev/null 2>&1 && ((COUNT++))
done < "$DOMAINS_FILE"
echo "$(date '+%H:%M:%S') $COUNT dominios refrescados" >> "$LOG"
```

**Cron (cada 20 min):** `*/20 * * * * /root/scripts/prewarm-dns.sh`

---

### Stack: indexers (Jackett + FlareSolverr)

**Directorios a crear:**
```bash
mkdir -p /root/docker/jackett/config /root/docker/jackett/downloads
```

```yaml
services:
  jackett:
    image: lscr.io/linuxserver/jackett:latest
    container_name: jackett
    environment:
      - PUID=0
      - PGID=0
      - TZ=Europe/Madrid
      - AUTO_UPDATE=true
    volumes:
      - /root/docker/jackett/config:/config
      - /root/docker/jackett/downloads:/downloads
    ports:
      - 9117:9117
    restart: unless-stopped

  flaresolverr:
    image: flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - LOG_LEVEL=info
      - LOG_HTML=false
      - CAPTCHA_SOLVER=none
      - TZ=Europe/Madrid
    ports:
      - "8191:8191"
    restart: unless-stopped
```

---

### Stack: uptime-kuma

**Directorios a crear:**
```bash
mkdir -p /opt/uptime-kuma
```

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    dns:
      - 192.168.50.67   # AdGuard (primario)
      - 8.8.8.8         # Google (fallback)
      - 1.1.1.1         # Cloudflare (fallback)
    ports:
      - 3001:3001
    volumes:
      - /opt/uptime-kuma:/app/data
      - /var/run/docker.sock:/var/run/docker.sock:ro
    restart: unless-stopped
```

**Función:** Monitorización de disponibilidad de servicios (up/down, latencia, notificaciones).

---

### Resumen de servicios

| Nombre | Stack | Puerto | Función |
|--------|-------|--------|---------|
| **portainer** | (docker run) | 9000 | Gestión Docker |
| **homeassistant** | home-assistant | 8123 | Domótica |
| **adguard** | adguard | 3000/53 | DNS + Bloqueador ads |
| **jackett** | indexers | 9117 | Indexador torrents |
| **flaresolverr** | indexers | 8191 | Bypass Cloudflare |
| **uptime-kuma** | uptime-kuma | 3001 | Monitorización |

---

# 🌐 Accesos Web

| Servicio | Ubicación | URL |
|----------|-----------|-----|
| **Proxmox** | Nodo | https://192.168.50.8:8006 |
| **Vaultwarden** | LXC 101 | https://192.168.50.148:8000 |
| **Portainer** | LXC 100 | http://192.168.50.67:9000 |
| **Uptime Kuma** | LXC 100 | http://192.168.50.67:3001 |
| **Home Assistant** | LXC 100 | http://192.168.50.67:8123 |
| **AdGuard** | LXC 100 | http://192.168.50.67:3000 |
| **Jackett** | LXC 100 | http://192.168.50.67:9117 |
| **FlareSolverr** | LXC 100 | http://192.168.50.67:8191 |

> **Nota:** La IP de LXC 100 (`192.168.50.67`) está ligada a la MAC `BC:24:11:AF:E8:2E` en el router.

---

# 🚀 Próximas mejoras

Para ver las mejoras planificadas y cosas por hacer, consulta:

📄 **[TODO.md](./TODO.md)**
