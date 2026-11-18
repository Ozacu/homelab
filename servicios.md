# Servicios

Cada servicio corre en su propio contenedor LXC sin privilegios. Los que necesitan Docker llevan `nesting=1`.

## Patrón base

```bash
# Crear el CT desde la interfaz, luego:
pct set <ID> -features nesting=1
pct start <ID>
pct enter <ID>

apt update && apt install -y curl gnupg locales
sed -i 's/# en_US.UTF-8/en_US.UTF-8/' /etc/locale.gen
locale-gen && update-locale LANG=en_US.UTF-8
curl -fsSL https://get.docker.com | sh
```

Después se agrega en Nginx Proxy Manager con su nombre bajo `*.lab.ozacu.com`.

---

## Jellyfin con GPU

El reto está en compartir la RTX 3050 con un contenedor sin privilegios. La ruta correcta no es passthrough a VM, que en laptops con gráficos híbridos casi nunca funciona, sino pasar los dispositivos a un LXC.

**En el host:**

```bash
# Driver con módulos MIT/GPL (recomendados para Turing y posteriores)
apt install -y proxmox-headers-$(uname -r) proxmox-default-headers \
  build-essential dkms pkg-config libglvnd-dev

echo -e "blacklist nouveau\noptions nouveau modeset=0" \
  > /etc/modprobe.d/blacklist-nouveau.conf
update-initramfs -u && reboot

./NVIDIA-Linux-x86_64-<version>.run --dkms

# Que los dispositivos existan al arrancar
printf 'nvidia\nnvidia_uvm\n' > /etc/modules-load.d/nvidia.conf
```

El paquete `proxmox-default-headers` es importante: hace que DKMS recompile el driver cuando Proxmox actualiza el kernel.

**En el contenedor**, pasar estos cuatro dispositivos desde Resources:

```
/dev/nvidia0
/dev/nvidiactl
/dev/nvidia-uvm
/dev/nvidia-uvm-tools
```

**Instalar el driver dentro**, con la misma versión del host:

```bash
pct push <ID> /root/NVIDIA-Linux-x86_64-<version>.run /root/nvidia.run
sh /root/nvidia.run --no-kernel-module
nvidia-smi
```

La bandera `--no-kernel-module` es la clave. El contenedor usa el módulo del host, solo necesita las librerías.

---

## Ollama con Open WebUI

Mismo passthrough de GPU, más el toolkit de NVIDIA para Docker.

En LXC sin privilegios hay que desactivar la gestión de cgroups del toolkit, porque intenta manipular reglas del host:

```bash
# /etc/nvidia-container-runtime/config.toml
no-cgroups = true
```

Y entonces los dispositivos se listan explícitamente en el compose:

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    restart: unless-stopped
    environment:
      - OLLAMA_HOST=0.0.0.0
    volumes:
      - ./ollama:/root/.ollama
    devices:
      - /dev/nvidia0:/dev/nvidia0
      - /dev/nvidiactl:/dev/nvidiactl
      - /dev/nvidia-uvm:/dev/nvidia-uvm
      - /dev/nvidia-uvm-tools:/dev/nvidia-uvm-tools
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

  webui:
    image: ghcr.io/open-webui/open-webui:main
    restart: unless-stopped
    ports:
      - '3000:8080'
    volumes:
      - ./webui:/app/backend/data
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - ENABLE_OPENAI_API=false
      - RAG_EMBEDDING_ENGINE=ollama
    depends_on:
      - ollama
```

`RAG_EMBEDDING_ENGINE=ollama` evita que Open WebUI descargue sus propios modelos de embeddings. Sin esa variable bajó cerca de 900 MB sin que nadie lo usara.

Con 4 GB de VRAM caben modelos de 3B cómodos y de 7B cuantizados. Más grandes reparten capas con la CPU y la velocidad se desploma. Se verifica con `ollama ps`, columna PROCESSOR.

---

## AMP

Instalación nativa, sin Docker. Correr Docker dentro de un LXC sin privilegios para aislar servidores de juego es redundante: el LXC ya aísla.

```bash
bash <(wget -qO- getamp.sh)
```

Respuestas durante la instalación:

| Pregunta | Respuesta | Por qué |
|---|---|---|
| Enable HTTPS | No | Lo maneja el reverse proxy |
| Install Docker | No | El LXC ya aísla |
| Install Java | Sí | Necesario para Minecraft |

ZeroTier se instala dentro del mismo contenedor. Requiere `nesting=1` y el dispositivo TUN:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

---

## n8n

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    restart: unless-stopped
    ports:
      - '5678:5678'
    volumes:
      - n8n_data:/home/node/.n8n
    environment:
      - N8N_HOST=n8n.lab.ozacu.com
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.lab.ozacu.com/
      - GENERIC_TIMEZONE=America/Vancouver
      - TZ=America/Vancouver
      - N8N_SECURE_COOKIE=true

volumes:
  n8n_data:
```

Usar un volumen de Docker y no un bind mount. n8n corre como UID 1000 y en LXC sin privilegios los UIDs no coinciden con el host, lo que produce `EACCES` al arrancar.

Las variables de dominio hay que ponerlas antes del primer arranque. n8n guarda la URL base y los webhooks quedan mal si se cambia después.

### Flujo de keepalive para Supabase

El plan gratuito de Supabase pausa la base tras una semana sin actividad. Un flujo simple en n8n lo evita.

1. Schedule Trigger, cada 2 días.
2. HTTP Request, método GET, a `https://<proyecto>.supabase.co/rest/v1/<tabla>?select=id&limit=1`.
3. Headers `apikey` y `Authorization: Bearer <anon key>`. Se usa la anon key, nunca la service role.

Después de probarlo hay que publicar el workflow. Sin eso, el schedule no corre solo.

---

## Actual Budget

```yaml
services:
  actual:
    image: ghcr.io/actualbudget/actual-server:latest
    restart: unless-stopped
    ports:
      - '5006:5006'
    volumes:
      - actual_data:/data
    environment:
      - TZ=America/Vancouver

volumes:
  actual_data:
```

Sincronización bancaria por SimpleFIN Bridge.

---

## Homepage

```yaml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    restart: unless-stopped
    ports:
      - '3000:3000'
    volumes:
      - ./config:/app/config
    environment:
      - HOMEPAGE_ALLOWED_HOSTS=home.lab.ozacu.com
```

Sin `HOMEPAGE_ALLOWED_HOSTS` bloquea el acceso por dominio.

---

## ntfy y Uptime Kuma

Alertas al teléfono y monitoreo de todos los servicios. Corren juntos en un solo contenedor (CT 206) en el Beelink. Cuando llegue la Raspberry Pi, se mudan allá. Un monitor no debería vivir en la misma máquina que vigila.

```yaml
services:
  ntfy:
    image: binwiederhier/ntfy:latest
    command: serve
    restart: unless-stopped
    ports:
      - '8080:80'
    volumes:
      - ntfy_data:/var/lib/ntfy
    environment:
      - TZ=America/Vancouver
      - NTFY_BASE_URL=https://ntfy.lab.ozacu.com
      - NTFY_BEHIND_PROXY=true
      - NTFY_CACHE_FILE=/var/lib/ntfy/cache.db
      - NTFY_AUTH_FILE=/var/lib/ntfy/user.db
      - NTFY_AUTH_DEFAULT_ACCESS=deny-all

  kuma:
    image: louislam/uptime-kuma:2
    restart: unless-stopped
    ports:
      - '3001:3001'
    volumes:
      - kuma_data:/app/data
    environment:
      - TZ=America/Vancouver
    extra_hosts:
      - "home.lab.ozacu.com:192.168.8.53"

volumes:
  ntfy_data:
  kuma_data:
```

Crear el usuario administrador de ntfy:

```bash
docker compose exec ntfy ntfy user add --role=admin <usuario>
```

### Decisiones

**`deny-all` en ntfy.** Nadie publica ni lee sin usuario, aunque llegue a la URL.

**El tema es parte de la seguridad.** Se usa un nombre difícil de adivinar, no `alertas` ni `homelab`.

**`NTFY_UPSTREAM_BASE_URL=https://ntfy.sh`** solo hace falta en iPhone. iOS no mantiene conexiones abiertas, así que los avisos instantáneos pasan por los servidores de Apple. Con esta variable, ntfy.sh solo recibe el aviso de que hay un mensaje nuevo. El texto sigue en el servidor propio. En Android no se necesita.

**Kuma y ntfy hablan por la red interna de Docker.** En Kuma la URL del servidor es `http://ntfy:80`, no el dominio público. Así las alertas no dependen del proxy ni de Tailscale.

**`extra_hosts` en Kuma.** Los dominios `*.lab.ozacu.com` resuelven a una IP de Tailscale, y el contenedor no tiene Tailscale. Con `extra_hosts` se le dice a Kuma que ese nombre apunta a la IP local del proxy. El certificado se valida igual.

### Monitores

Se usan IPs para los chequeos principales. Si uno falla, se sabe si cayó el servicio o solo el proxy.

| Monitor | Tipo | Destino |
|---|---|---|
| Proxmox laptop | TCP | `192.168.8.188:8006` |
| Proxmox Beelink | TCP | `192.168.8.50:8006` |
| NAS | Ping | `192.168.8.125` |
| PBS | TCP | `192.168.8.190:8007` |
| Jellyfin | HTTP | `http://192.168.8.51:8096` |
| AMP | HTTP | `http://192.168.8.60:8080` |
| Open WebUI | HTTP | `http://192.168.8.54:3000` |
| n8n | HTTP | `http://192.168.8.62:5678` |
| Actual Budget | HTTP | `http://192.168.8.63:5006` |
| Proxy y certificado | HTTP | `https://home.lab.ozacu.com` |

Intervalo de 60 segundos y 3 reintentos antes de alertar, para que un parpadeo no genere un aviso. En el monitor del proxy se activa el aviso de vencimiento de certificado, como red de seguridad si falla la renovación.

### Qué no cubre

Si se va la luz, el router también cae y ningún aviso sale de casa. Eso solo lo resuelve un UPS. Y mientras el monitor viva en el Beelink, si el Beelink cae nadie avisa.

### Probar la alerta

Una alerta que nunca se probó no sirve cuando hace falta. Se detiene un contenedor a propósito, se espera el tiempo de los reintentos y se confirma que llega el aviso de caída y el de recuperación.

```bash
pct stop <ID>
pct start <ID>
```

---

## Samba

Para copiar archivos a la media sin pelear con permisos:

```ini
[media]
   path = /media
   browseable = yes
   writable = yes
   valid users = <usuario>
   force user = root
   create mask = 0664
   directory mask = 0775
```

`force user = root` hace que todo lo copiado quede con el dueño correcto en el host, que en un LXC sin privilegios es el UID 100000.
