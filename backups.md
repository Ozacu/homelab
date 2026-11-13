# Backups

## Estrategia

Cuatro capas, cada una cubre un fallo distinto.

| Capa | Qué protege | Recuperación |
|---|---|---|
| Snapshots de Proxmox | Error propio al configurar | Segundos |
| PBS en HDD local | Falla del disco del nodo | Minutos |
| Sincronización al NAS | Falla del disco de backup | Minutos |
| RAID 1 del NAS | Falla de un disco del NAS | Automática |

Lo que **no** cubre: robo, incendio, inundación. Todo está en la misma habitación. Falta una copia fuera de casa para lo irreemplazable.

Vale aclarar algo que se confunde seguido: **RAID 1 no es un backup.** Protege contra la muerte de un disco y nada más. Un borrado accidental se replica al espejo de inmediato.

## Proxmox Backup Server

PBS corre en un contenedor **privilegiado** en el Beelink. Privilegiado porque maneja permisos de archivo que dan problemas sin privilegios.

```bash
curl -fsSL https://enterprise.proxmox.com/debian/proxmox-release-trixie.gpg \
  -o /etc/apt/trusted.gpg.d/proxmox-release-trixie.gpg
echo "deb http://download.proxmox.com/debian/pbs trixie pbs-no-subscription" \
  > /etc/apt/sources.list.d/pbs.list
apt update && apt install -y proxmox-backup-server
```

El HDD se monta en el host y se pasa al contenedor:

```bash
# /etc/fstab
UUID=<uuid>  /mnt/backups  ext4  defaults,nofail  0  2
```

```bash
pct set 201 -mp0 /mnt/backups,mp=/datastore
```

La opción `nofail` importa: si el disco no está conectado, el nodo arranca igual en vez de quedarse colgado.

Los dos nodos se conectan a PBS como storage desde **Datacenter > Storage > Add > Proxmox Backup Server**, usando la huella que devuelve:

```bash
proxmox-backup-manager cert info | grep -i fingerprint
```

## Jobs de backup

Diario a las 3 AM, modo snapshot, con `Selection mode: All` para que los contenedores nuevos entren solos.

Retención: 7 diarios, 4 semanales, 6 mensuales. PBS deduplica, así que varias versiones ocupan poco más que una.

Dos cosas que se olvidan:

1. **Marcar la casilla Enable.** Un job sin habilitar existe pero nunca corre.
2. **Probar una restauración.** Un backup que nunca se restauró es una esperanza, no un respaldo.

## Sincronización al NAS

La carpeta NFS del NAS se monta en el host del Beelink y se pasa a PBS como segundo datastore:

```bash
# /etc/fstab
192.168.8.125:/backups-proxmox  /mnt/nas-pbs  nfs  vers=3,hard,timeo=600,retrans=2,nofail,_netdev  0  0
```

```bash
pct set 201 -mp1 /mnt/nas-pbs,mp=/nas,backup=0
```

En PBS se registra como datastore `nas` y se crea un **Sync Job** con dirección pull: el datastore destino tira del origen. Diario a las 5 AM, dos horas después del backup.

`Remove Vanished` queda desmarcado a propósito. Así el NAS conserva más historia que el disco local cuando la retención borra snapshots viejos.

## Montajes NFS: usar hard, no soft

Esto costó un error de datos.

Con `soft`, el cliente NFS se rinde si el servidor tarda en responder. En un NAS ARM con 1 GB de RAM, vaciar la caché de un archivo de 17 GB tarda más de lo que `soft` espera, y la copia falla al cerrar con `Input/output error`. Peor aún, `soft` puede dejar archivos corruptos sin avisar.

Opciones correctas para escritura:

```
vers=3,hard,timeo=600,retrans=2,nofail,_netdev
```

Con `hard` el cliente espera en vez de rendirse. Se notan pausas al final de archivos grandes, y eso es exactamente lo que evita la corrupción.

## Verificar antes de borrar

Al mover datos, comparar tamaños no basta. Comparar hashes sí:

```bash
rsync -avh --progress /origen/ /destino/
md5sum /origen/archivo /destino/archivo
```

Y nunca usar `mv` para mover a través de la red. Si se corta a medias, quedan archivos partidos sin copia de respaldo.
