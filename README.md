# 🖥️ Proxmox VE Configuration - geekcon

Configuration documentation and scripts for my Proxmox VE home server.

## 📋 Overview

This repository contains the complete configuration for reinstalling and managing my Proxmox server from scratch.

### Hardware

- **CPU:** AMD Ryzen AI 9 HX 370 (12 cores / 24 threads)
- **RAM:** 30 GB DDR5
- **Storage:** 2x NVMe (1.8TB system + 1.9TB backups)
- **GPU:** AMD Radeon 890M (integrated)
- **NPU:** AMD XDNA (AI accelerator)

### What's Running

| Type | Name | Description |
|------|------|-------------|
| **LXC 100** | docker-commander | Docker host (Home Assistant, AdGuard, Jackett, etc.) |
| **LXC 101** | vaultwarden | Password manager (Bitwarden-compatible) |
| **VM 200** | windows11 | Windows 11 Pro (RDP enabled) |

## 📚 Documentation

| File | Description |
|------|-------------|
| [proxmox-config.md](proxmox-config.md) | Complete configuration reference |
| [QUICKSTART.md](QUICKSTART.md) | Step-by-step reinstallation checklist |
| [TODO.md](TODO.md) | Planned improvements |
| [AGENTS.md](AGENTS.md) | AI assistant system prompt |

## 🔧 Key Features Documented

- APT repositories (no-subscription setup)
- CPU optimization with `tuned` (Ryzen profile)
- Thermal monitoring script
- Wake-on-LAN configuration
- Subscription nag removal
- GPU/NPU passthrough for LXC
- Docker services via Portainer stacks
- DNS cache prewarm script
- Backup and restore procedures

## ⚠️ Note

Some configurations contain placeholder values (passwords, IPs) that need to be replaced with your actual values.

## 📜 License

Personal documentation. Feel free to use as reference for your own setup.
