# Homelab

Infraestructura personal de self-hosting sobre Proxmox VE, con tres nodos, almacenamiento en red y acceso remoto sin exponer puertos a internet.

Este repositorio documenta la arquitectura, las decisiones de diseño y los problemas que tuve que resolver durante la construcción.

## Qué corre aquí

| Categoría | Servicios |
|---|---|
| Virtualización | Proxmox VE 9 en dos nodos |
| Backups | Proxmox Backup Server con sincronización a NAS |
| Media | Jellyfin con transcodificación por GPU |
| Juegos | AMP con servidores de juego |
| IA local | Ollama y Open WebUI sobre RTX 3050 |
| Automatización | n8n |
| Finanzas | Actual Budget |
| Red | Nginx Proxy Manager, Tailscale, ZeroTier |
| Monitoreo | Uptime Kuma con alertas por ntfy, en una Raspberry Pi independiente |
| Dashboard | Homepage |

## Hardware

| Equipo | Especificaciones | Rol |
|---|---|---|
| Lenovo IdeaPad Gaming 3 | Ryzen 5 6600H, 32 GB DDR5, RTX 3050 4 GB, NVMe 256 GB + 1 TB | Nodo principal. GPU y cargas pesadas |
| Beelink SER5 | Ryzen 5 5500U, 32 GB DDR4, NVMe 500 GB | Nodo de servicios y juegos |
| QNAP TS-231P | ARM 32 bits, 1 GB RAM, 2x3 TB en RAID 1 | Almacenamiento en red |
| WD My Passport 1 TB | USB 3.0 | Destino de backups |
| Raspberry Pi 5 8 GB | NVMe 500 GB (placa N04), carcasa metálica con ventilador activo, fuente de 27 W | Nodo centinela. Alertas y monitoreo, fuera de los otros dos nodos |
| GL-iNet GL-MT3000 | Modo repetidor WiFi | Router de la subred |

## Decisiones de diseño

**Una laptop como nodo principal.** La batería funciona como respaldo ante cortes de luz, que es justo lo que falta sin UPS. Además tiene GPU, que el mini PC no.

**Dos nodos independientes, no un clúster.** Un clúster de dos nodos tiene problemas de quórum: si uno cae, el otro se bloquea. Separados son más simples y más resistentes.

**Juegos aislados en su propio nodo.** El contenedor de AMP es el único al que entran personas externas por ZeroTier. Vive en el Beelink, lejos de los servicios personales.

**Nada expuesto a internet.** Todo el acceso remoto pasa por Tailscale. Los certificados se emiten por validación DNS, así que nunca hace falta abrir un puerto.

**Un nodo pequeño que vigila a los grandes.** El monitoreo y las alertas viven en una Raspberry Pi. Un monitor que corre en la máquina que vigila no avisa cuando esa máquina cae. La Pi consume poco y no depende de Proxmox, así que sigue en pie si los otros dos fallan.

**LXC sobre VM siempre que sea posible.** Consumen menos y arrancan más rápido. Las VMs quedan para lo que necesite kernel propio.

## Documentación

- [Arquitectura](arquitectura.md)
- [Servicios](servicios.md)
- [Backups](backups.md)
- [Problemas resueltos](troubleshooting.md)

## Estado

En construcción. La Raspberry Pi ya corre ntfy y Uptime Kuma. Pendiente: AdGuard Home y Vaultwarden en la Pi, copia fuera de casa, Immich y Pocket ID.
