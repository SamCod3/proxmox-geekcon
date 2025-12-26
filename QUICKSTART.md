# 🚀 Guía Rápida de Reinstalación - Proxmox geekcon

> **Usa este checklist tras una instalación limpia de Proxmox VE**

---

## ✅ Checklist de configuración

### 1. Repositorios (sin suscripción)

```bash
# Deshabilitar enterprise
mv /etc/apt/sources.list.d/pve-enterprise.sources /etc/apt/sources.list.d/pve-enterprise.sources.disabled

# Habilitar no-subscription
cat > /etc/apt/sources.list.d/pve-no-subscription.sources << 'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF

# Ceph (opcional)
cat > /etc/apt/sources.list.d/ceph.sources << 'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/ceph-squid
Suites: trixie
Components: no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF

apt update && apt upgrade -y
```

---

### 2. Montar disco de backups

```bash
mkdir -p /mnt/backups
UUID_BACKUP=$(blkid -s UUID -o value /dev/nvme0n1)
echo "UUID=$UUID_BACKUP /mnt/backups ext4 defaults 0 2" >> /etc/fstab
mount -a

# Registrar en Proxmox
pvesm add dir backup-storage --path /mnt/backups --content vztmpl,backup,snippets,iso
```

---

### 3. Quitar aviso de suscripción

```bash
cat > /usr/local/bin/pve-remove-nag.sh << 'SCRIPT'
#!/bin/sh
WEB_JS=/usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
if [ -s "$WEB_JS" ] && ! grep -q NoMoreNagging "$WEB_JS"; then
    sed -i -e "/data\.status/ s/!//" -e "/data\.status/ s/active/NoMoreNagging/" "$WEB_JS"
fi
SCRIPT

chmod +x /usr/local/bin/pve-remove-nag.sh
echo 'DPkg::Post-Invoke { "/usr/local/bin/pve-remove-nag.sh"; };' > /etc/apt/apt.conf.d/no-nag-script
/usr/local/bin/pve-remove-nag.sh
systemctl restart pveproxy
```

---

### 4. Tuned (optimización CPU Ryzen)

```bash
apt install tuned -y
mkdir -p /etc/tuned/profiles/proxmox-ryzen

cat > /etc/tuned/profiles/proxmox-ryzen/tuned.conf << 'EOF'
[main]
include=virtual-host

[cpu]
governor=powersave
energy_performance_preference=balance_performance
EOF

tuned-adm profile proxmox-ryzen
```

---

### 5. Monitorización térmica

```bash
apt install lm-sensors -y

cat > /usr/local/bin/temp-watchdog.sh << 'EOF'
#!/bin/bash
TEMP=$(sensors -u k10temp-pci-00c3 | grep "temp1_input" | awk '{print $2}')
TEMP_INT=${TEMP%.*}
if [ "$TEMP_INT" -ge 95 ]; then
    echo "CRITICAL: CPU at ${TEMP}°C" | logger -t temp-watchdog -p user.emerg
elif [ "$TEMP_INT" -ge 85 ]; then
    echo "WARNING: CPU at ${TEMP}°C" | logger -t temp-watchdog -p user.alert
fi
EOF

chmod +x /usr/local/bin/temp-watchdog.sh
echo '* * * * * root /usr/local/bin/temp-watchdog.sh' > /etc/cron.d/temp-watchdog
```

---

### 6. Crear contenedor LXC docker-commander

```bash
# Buscar última versión del template Debian 13
pveam update
pveam available | grep "debian-13-standard"
# Ejemplo: debian-13-standard_13.1-2_amd64.tar.zst

# Descargar en backup-storage (persiste tras reinstalar)
pveam download backup-storage debian-13-standard_13.1-2_amd64.tar.zst

# Crear contenedor (MAC fija para IP estática del router)
# Ajustar nombre del template si hay versión más nueva
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

# Configuración avanzada (GPU + CPU Pinning)
cat >> /etc/pve/lxc/100.conf << 'EOF'
lxc.cgroup2.cpuset.cpus: 8-23
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.cgroup2.devices.allow: c 234:0 rwm
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
lxc.mount.entry: /dev/kfd dev/kfd none bind,optional,create=file
EOF

pct start 100
```

---

### 7. Instalar Docker en el contenedor

```bash
pct enter 100

apt update && apt upgrade -y
apt install -y curl gnupg lsb-release ca-certificates
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Instalar Portainer
docker volume create portainer_data
docker run -d --name portainer --restart unless-stopped \
  -p 9000:9000 -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest

exit
```

---

### 8. Crear stacks en Portainer

Acceder a http://192.168.50.67:9000 y crear los stacks:

- **home-assistant** → Ver `proxmox-config.md`
- **adguard** → Ver `proxmox-config.md`
- **indexers** (Jackett + FlareSolverr) → Ver `proxmox-config.md`

---

## 🌐 URLs finales

| Servicio | URL |
|----------|-----|
| Proxmox | https://192.168.50.8:8006 |
| Portainer | http://192.168.50.67:9000 |
| Home Assistant | http://192.168.50.67:8123 |
| AdGuard | http://192.168.50.67:3000 |
| Jackett | http://192.168.50.67:9117 |

---

📄 **Documentación completa:** [proxmox-config.md](./proxmox-config.md)
