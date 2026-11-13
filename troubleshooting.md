# Problemas resueltos

Cada uno de estos costó tiempo. Los dejo documentados con la causa real, no solo la solución.

---

## apt no resuelve nombres en Proxmox recién instalado

**Síntoma:** `Temporary failure resolving 'deb.debian.org'`

**Causa:** dos cosas distintas que se ven igual. Primero, el gateway apuntaba a una IP donde no hay nadie. Segundo, falta el DNS.

**Diagnóstico:**

```bash
ping -c2 1.1.1.1        # si falla, es red o gateway
ping -c2 deb.debian.org # si solo falla este, es DNS
ip r                    # revisar la ruta por defecto
```

**Solución:**

```bash
ip route replace default via <gateway> dev vmbr0   # prueba en caliente
nano /etc/network/interfaces                        # y permanente
echo -e "nameserver 1.1.1.1\nnameserver 8.8.8.8" > /etc/resolv.conf
```

---

## Repositorio enterprise sin suscripción

**Síntoma:** `401 Unauthorized` en `enterprise.proxmox.com`

**Causa:** la instalación por defecto apunta a los repos de pago.

**Solución:** en **Updates > Repositories**, desactivar `pve-enterprise` y `ceph` enterprise, y agregar `No-Subscription`. En Proxmox 9 los repos son archivos `.sources`, no `.list`.

---

## El instalador de NVIDIA no encuentra los headers

**Síntoma:** `Unable to find the kernel source tree`

**Causa:** el nombre del paquete. En Proxmox 8 y 9 es `proxmox-headers`, no `pve-headers`.

**Solución:**

```bash
apt install -y proxmox-headers-$(uname -r) proxmox-default-headers
ls /lib/modules/$(uname -r)/build   # verificar
```

Si no encuentra el paquete para el kernel actual, hay uno más nuevo esperando. Reiniciar y repetir.

---

## Secure Boot bloquea el módulo de NVIDIA

**Síntoma:** el instalador pregunta por firma de módulos. Instalar sin firmar deja el driver sin cargar.

**Causa:** la laptop tiene Secure Boot activado y el kernel rechaza módulos sin firma de confianza.

**Opciones:**

1. Desactivar Secure Boot en BIOS. Es lo práctico para un servidor doméstico.
2. Registrar la llave de DKMS en el firmware:

```bash
mokutil --import /var/lib/dkms/mok.pub
```

La segunda requiere estar físicamente frente al equipo para completar el MOK Manager al reiniciar.

---

## Los dispositivos /dev/nvidia* no existen

**Síntoma:** `ls /dev/nvidia*` no devuelve nada, aunque el driver esté instalado.

**Causa:** los nodos de dispositivo se crean cuando algo toca el driver por primera vez.

**Solución:**

```bash
nvidia-smi
nvidia-modprobe -c0 -u
printf 'nvidia\nnvidia_uvm\n' > /etc/modules-load.d/nvidia.conf
```

Verificar después de reiniciar, antes de ejecutar nada más. Si no aparecen solos, crear un servicio de systemd que los genere al arranque.

---

## El toolkit de NVIDIA falla en LXC sin privilegios

**Síntoma:**

```
nvidia-container-cli: mount error: failed to add device rules:
bpf_prog_query(BPF_CGROUP_DEVICE) failed: operation not permitted
```

**Causa:** el toolkit intenta manipular reglas de cgroup del host, algo que un contenedor sin privilegios no puede hacer.

**Solución:** en `/etc/nvidia-container-runtime/config.toml`:

```
no-cgroups = true
```

Y listar los dispositivos explícitamente en el `docker-compose.yml`, porque ya no los agrega solo.

---

## gpg no está instalado y falla la llave del repositorio

**Síntoma:** `gpg: command not found`, seguido de `Failed to parse keyring` y `repository is not signed`.

**Causa:** la plantilla mínima de Debian no trae `gnupg`. El archivo de llave nunca se creó y todo lo demás falló en cadena.

**Solución:**

```bash
apt install -y gnupg ca-certificates
```

Verificar que el archivo existe antes de seguir, en vez de encadenar comandos a ciegas.

---

## El contenedor LXC no tiene DNS

**Síntoma:** los mismos errores de resolución que en el host, pero dentro del contenedor.

**Causa:** el contenedor heredó una configuración de DNS que no funciona.

**Solución:**

```bash
pct set <ID> -nameserver 1.1.1.1
pct reboot <ID>
```

Mejor aún: ponerlo en el formulario de creación, pestaña DNS.

---

## n8n en bucle de reinicio

**Síntoma:** `EACCES: permission denied, open '/home/node/.n8n/config'`

**Causa:** n8n corre como UID 1000. En un LXC sin privilegios los UIDs están desplazados, así que un bind mount creado por root no es escribible.

**Solución:** usar un volumen de Docker en vez de un bind mount. Docker gestiona los permisos por su cuenta.

---

## Puerto ocupado después de mover un compose

**Síntoma:** `Bind for 0.0.0.0:3000 failed: port is already allocated`

**Causa:** mover el `docker-compose.yml` a otra carpeta cambia el nombre del proyecto. Los contenedores viejos siguen vivos bajo el nombre anterior.

**Solución:**

```bash
docker ps -a                    # ver quién sigue vivo
docker rm -f <nombre-viejo>
docker network rm <red-vieja>
```

Y si un contenedor quedó creado sin el mapeo de puertos:

```bash
docker compose up -d --force-recreate
```

---

## Snapshots bloqueados en un contenedor

**Síntoma:** "The current guest configuration does not support taking new snapshots"

**Causa:** un bind mount apuntando a un directorio del host. Proxmox no sabe cómo versionarlo.

**Solución:** marcarlo para que lo ignore.

```bash
pct set <ID> -mp0 /ruta/host,mp=/ruta/contenedor,backup=0
```

---

## Unidad de red montada que no aparece en el Explorador

**Síntoma:** `net use` dice que funcionó, pero no hay unidad visible en Windows.

**Causa:** se montó desde un CMD elevado. Windows separa las sesiones: lo que monta el administrador no lo ve el Explorador.

**Solución:** montar desde un CMD normal. O, para que ambas sesiones compartan unidades:

```
reg add HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v EnableLinkedConnections /t REG_DWORD /d 1 /f
```

---

## El dominio resuelve pero no carga

**Síntoma:** `ERR_CONNECTION_TIMED_OUT` en un equipo, mientras funciona en otro.

**Causa:** los registros DNS apuntan a una IP de Tailscale. Un equipo sin Tailscale resuelve el nombre pero no sabe llegar.

**Solución:** instalar Tailscale en ese equipo. Esto no es un error, es la protección funcionando.

---

## Jellyfin deja de cargar tras cambiar la Base URL

**Síntoma:** "Servidor no disponible" después de configurar un subdirectorio.

**Causa:** el navegador guarda la dirección del servidor en almacenamiento local.

**Solución:** probar en incógnito para confirmar, luego limpiar los datos del sitio.

Mejor todavía: no usar Base URL. Rompe los clientes de TV y móvil. Darle un puerto propio en el reverse proxy en vez de un subdirectorio.
