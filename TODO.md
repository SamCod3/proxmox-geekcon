# 🚀 Próximas Mejoras - Proxmox geekcon

> **Estado:** Planificado  
> **Última actualización:** 26 Diciembre 2025

---

## 📋 Por hacer

### Alta prioridad

- [ ] **🔐 Mejorar seguridad: Dejar de usar root**

  **Situación actual:**
  - Proxmox: Acceso directo como root
  - LXC 100: Todo se ejecuta como root
  - Docker: Contenedores ejecutándose como root

  **Plan de acción:**

  #### 1. Proxmox (Nodo) - Crear usuario administrador
  ```bash
  # Crear usuario en Proxmox
  pveum user add admin@pam
  pveum passwd admin@pam
  
  # Asignar rol de administrador
  pveum acl modify / -user admin@pam -role Administrator
  
  # Opcional: Configurar sudo en el sistema
  apt install sudo
  useradd -m -s /bin/bash admin
  usermod -aG sudo admin
  passwd admin
  ```
  
  #### 2. LXC 100 - Crear usuario no-root
  ```bash
  pct enter 100
  
  # Crear usuario
  useradd -m -s /bin/bash dockeruser
  passwd dockeruser
  
  # Añadir al grupo docker (para ejecutar docker sin sudo)
  usermod -aG docker dockeruser
  
  # Crear directorio para datos de Docker
  mkdir -p /home/dockeruser/docker-data
  chown -R dockeruser:dockeruser /home/dockeruser/docker-data
  ```
  
  #### 3. Docker - Ejecutar contenedores como non-root
  ```yaml
  # En cada stack de Portainer, añadir:
  services:
    myservice:
      user: "1000:1000"  # UID:GID del dockeruser
      # O para servicios que lo soporten:
      environment:
        - PUID=1000
        - PGID=1000
  ```
  
  #### 4. SSH - Mantener root pero con usuario alternativo
  ```bash
  # En /etc/ssh/sshd_config del nodo, mantener:
  PermitRootLogin yes
  
  # Pero ahora también puedes usar el usuario admin
  # Root sigue disponible para emergencias
  ```
  
  **Servicios afectados y cambios necesarios:**
  
  | Servicio | Cambio |
  |----------|--------|
  | Portainer | Mover socket a usuario o usar rootless Docker |
  | Home Assistant | Cambiar PUID/PGID o user |
  | AdGuard | Ya soporta non-root |
  | Jackett | Añadir PUID=1000, PGID=1000 |
  | FlareSolverr | Añadir user: 1000:1000 |
  
  **Consideraciones:**
  - GPU passthrough puede requerir permisos adicionales
  - Algunos servicios necesitan acceso a puertos < 1024
  - Hacer backup antes de los cambios

---

- [ ] **Backups automatizados**
  - Configurar vzdump para backups periódicos del contenedor
  - Retención: diaria (7), semanal (4), mensual (3)
  - Destino: `/mnt/backups`

- [ ] **IP estática para el contenedor**
  - Cambiar de DHCP a IP fija en el contenedor
  - O configurar reserva DHCP permanente en el router

### Media prioridad

- [ ] **Monitorización avanzada**
  - Instalar Prometheus + Grafana
  - Métricas de CPU, RAM, disco, temperatura
  - Alertas por Telegram/Discord

- [ ] **Más contenedores Docker**
  - [ ] Radarr / Sonarr (gestión de contenido)
  - [ ] qBittorrent / Transmission
  - [ ] Jellyfin / Plex (media server con aceleración GPU)
  - [ ] Nginx Proxy Manager (reverse proxy)
  - [ ] Vaultwarden (gestor contraseñas)

- [ ] **Certificados SSL**
  - Let's Encrypt para Proxmox web UI
  - Wildcard para servicios internos

### Baja prioridad

- [ ] **Cluster Proxmox**
  - Añadir segundo nodo
  - Alta disponibilidad (HA)

- [ ] **ZFS en lugar de LVM**
  - Migración a ZFS para snapshots y compresión
  - RAID-Z si se añaden más discos

- [ ] **VMs específicas**
  - [ ] Windows 11 con GPU passthrough para gaming
  - [ ] macOS Sonoma (Hackintosh VM)

---

## ✅ Completado

- [x] Instalación base Proxmox VE 9.1
- [x] Repositorios no-subscription configurados
- [x] Contenedor docker-commander con GPU passthrough
- [x] Tuned con perfil optimizado para Ryzen
- [x] Monitorización térmica
- [x] Wake-on-LAN
- [x] Script anti-nag

---

## 💡 Ideas futuras

- Integración con Home Assistant para control del servidor
- UPS con notificaciones de apagado seguro
- VPN (WireGuard) para acceso remoto
- Pi-hole/AdGuard como DNS de toda la red

---

## 📝 Notas

Añade aquí notas sobre configuraciones que quieras probar:

```
# Ejemplo:
# - Probar aceleración VAAPI en Jellyfin
# - Revisar configuración de IOMMU para más VMs con GPU
```
