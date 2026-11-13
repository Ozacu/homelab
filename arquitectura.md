# Arquitectura

## Diagrama lógico

```
Internet
   │
   ▼
Router de la casa (no administrado)
   │
   ▼
GL-iNet GL-MT3000 ── subred 192.168.8.0/24
   │
   ├── Switch 8 puertos
   │     ├── Laptop (Proxmox)      192.168.8.188
   │     ├── Beelink (Proxmox)     192.168.8.50
   │     └── NAS QNAP              192.168.8.125
   │
   └── Tailscale ──► acceso remoto desde cualquier lugar
```

El router principal de la casa no es administrable, así que el GL-iNet crea una subred propia en modo repetidor. Toda la infraestructura vive ahí.

## Reparto de servicios

El criterio es simple: lo que necesita GPU o RAM va a la laptop, lo que debe estar siempre encendido y es liviano va al Beelink.

### Nodo 1: Laptop

| ID | Servicio | IP |
|---|---|---|
| 101 | Nginx Proxy Manager | 192.168.8.53 |
| 301 | Jellyfin | 192.168.8.51 |
| 302 | Samba | 192.168.8.52 |
| 304 | Ollama + Open WebUI | 192.168.8.54 |

### Nodo 2: Beelink

| ID | Servicio | IP |
|---|---|---|
| 201 | Proxmox Backup Server | 192.168.8.190 |
| 203 | Homepage | 192.168.8.61 |
| 204 | n8n | 192.168.8.62 |
| 205 | Actual Budget | 192.168.8.63 |
| 401 | AMP + ZeroTier | 192.168.8.60 |

## Convención de IDs

| Rango | Uso |
|---|---|
| 1xx | Red e infraestructura |
| 2xx | Core: backups, SSO, dashboard |
| 3xx | Aplicaciones |
| 4xx | Juegos |
| 9xx | Laboratorio |

Definir esto antes de crear el primer contenedor evita tener que renombrar después.

## Almacenamiento

| Dónde | Qué guarda |
|---|---|
| Laptop, NVMe 256 GB | Sistema Proxmox |
| Laptop, NVMe 1 TB (ZFS, pool `tank`) | Contenedores y VMs |
| Beelink, NVMe 500 GB | Sistema y contenedores |
| Beelink, HDD 1 TB USB | Datastore de PBS |
| NAS, RAID 1 de 3 TB | Media, fotos y copia de backups |

El pool ZFS tiene la caché ARC limitada a 4 GB para dejar RAM a los contenedores:

```bash
echo "options zfs zfs_arc_max=4294967296" > /etc/modprobe.d/zfs.conf
update-initramfs -u
```

## Red y acceso remoto

**Tailscale** para acceso personal. Instalado en el contenedor del reverse proxy, no en cada servicio.

**Nginx Proxy Manager** como único punto de entrada. Cada servicio recibe un nombre bajo `*.lab.ozacu.com` con certificado wildcard de Let's Encrypt.

**Certificados por DNS-01** con Cloudflare. La validación ocurre creando un registro TXT temporal, así que nunca hace falta exponer el puerto 80. Los registros DNS apuntan a la IP de Tailscale, que no lleva a ningún lado desde internet.

**ZeroTier** aparte, solo dentro del contenedor de AMP, para que los amigos lleguen a los servidores de juego sin ver el resto de la red.

## Energía

Sin UPS por el momento. Las medidas en pie:

- La laptop tiene batería, que da entre una y dos horas de autonomía.
- Modo de conservación de batería activado, para que no se degrade estando enchufada 24/7.
- El Beelink arranca solo al volver la corriente, configurado en BIOS.
- Todo detrás de regletas con protección de sobretensión.

Para el modo de conservación en IdeaPad:

```bash
echo Long_Life > /sys/class/power_supply/BAT1/charge_types
echo 'w /sys/class/power_supply/BAT1/charge_types - - - - Long_Life' \
  > /etc/tmpfiles.d/battery-conservation.conf
```
