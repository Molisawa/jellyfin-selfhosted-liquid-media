# Despliegue en macOS

Cómo levantar el stack en una Mac con Docker Desktop, en Apple Silicon o Intel. Al terminar, sigue con
[configuracion.md](configuracion.md): apps, acceso remoto y mantenimiento.

Todos los valores de ejemplo (`example.com`, `192.168.1.50`, `<...>`) son placeholders: los reales van solo
en tu `.env`, que no se sube al repo.

---

## 1. Preparar la Mac

1. **Docker Desktop**: instálalo desde docker.com (elige Apple Silicon o Intel según tu Mac). En *Settings*:
   - *General → Start Docker Desktop when you sign in to your computer*.
   - *Resources*: al menos 4 GB de memoria.
2. **Que no se duerma**: en *Ajustes del Sistema → Batería* (o *Energía* en una Mac de escritorio), activa
   *Evitar el reposo automático cuando la pantalla esté apagada* con el cargador conectado. O por terminal:
   ```bash
   sudo pmset -c sleep 0 disksleep 0
   ```
   Una MacBook con la **tapa cerrada se duerme** a menos que esté conectada a corriente y a un monitor
   externo. Si va a ser servidor, déjala abierta o úsala con monitor.
3. **IP fija en la LAN**: reserva una IP para la Mac en el DHCP del router (ej. `192.168.1.50`). Mejor por
   cable.
4. **Firewall**: *Ajustes del Sistema → Red → Firewall*, activado. Cuando macOS pregunte, permite
   conexiones entrantes para Docker.

## 2. Clonar y configurar

```bash
git clone https://github.com/Molisawa/jellyfin-selfhosted-liquid-media.git
cd jellyfin-selfhosted-liquid-media
cp .env.example .env
```

Valores en `.env`:

| Variable | Valor | Por qué |
|---|---|---|
| `PUID` / `PGID` | `1000` (déjalo así) | Docker Desktop traduce los permisos; en Mac no importa el valor |
| `TZ` | ej. `America/Mexico_City` | horarios de tareas y logs |
| `MEDIA_PATH` | ver §3 | raíz única para hardlinks |
| `DOMAIN` | `localhost` para probar; tu dominio para acceso remoto | Caddy pide los certificados para `watch.` y `request.` |
| `ACME_EMAIL` | tu correo real (solo con dominio) | Let's Encrypt rechaza `example.com` |
| `LAN_BIND_IP` | `127.0.0.1` solo esta Mac, `0.0.0.0` toda la LAN | paneles admin visibles en la LAN, nunca en internet |
| `COMPOSE_PROFILES` | `novpn` o `vpn` | qBittorrent directo o detrás de Gluetun |

## 3. Almacenamiento y hardlinks

Los hardlinks solo funcionan si `torrents/` y `media/` están en **el mismo disco** y ese disco los soporta.

| Dónde | `MEDIA_PATH` | Hardlinks |
|---|---|---|
| Disco interno | `/Users/<tu-usuario>/MediaServer/data` | Sí |
| Disco externo en **APFS** o *Mac OS Plus* | `/Volumes/<disco>/data` | Sí |
| Disco externo en **exFAT** o FAT32 | — | **No**: cada importación ocupa el doble |
| Disco en NTFS | — | macOS solo lo lee; no sirve |

Si tu disco externo está en exFAT, reformatéalo a APFS con *Utilidad de Discos*, lo que **borra su
contenido**. Docker Desktop comparte `/Users` y `/Volumes` por defecto, así que no hace falta configurar
nada más.

Crea la estructura (ejemplo con disco externo):
```bash
mkdir -p /Volumes/<disco>/data/{torrents/{movies,tv},media/{movies,tv}}
```

**Comprueba los hardlinks** antes de configurar Sonarr y Radarr:
```bash
docker compose up -d sonarr
docker compose exec sonarr sh -c 'echo test > /data/torrents/tv/.hl && ln /data/torrents/tv/.hl /data/media/tv/.hl && stat -c "%h enlaces" /data/media/tv/.hl; rm -f /data/torrents/tv/.hl /data/media/tv/.hl'
```
Debe imprimir **`2 enlaces`**.

> Con disco externo: si se desconecta o se duerme, los contenedores verán la carpeta vacía. Mantenlo
> conectado y excluye esa carpeta de Time Machine si no quieres respaldar toda la media.

## 4. Levantar el stack

```bash
docker compose up -d
docker compose ps
```

**Transcodificación:** Docker en macOS corre dentro de una máquina virtual Linux sin acceso a la GPU de la
Mac, así que Jellyfin transcodifica **por CPU**. Un chip Apple Silicon aguanta un par de transcodificaciones
1080p a la vez; lo ideal es que tus clientes reproduzcan directo (*Direct Play*). Los overrides
`docker-compose.nvidia.yml` y `docker-compose.intel.yml` no aplican en Mac.

## 5. Probar en local (opcional)

Con `DOMAIN=localhost`, Caddy usa su propia autoridad certificadora y tienes:

- Jellyfin en `https://watch.localhost` y Seerr en `https://request.localhost`.
- Sonarr `localhost:8989` · Radarr `localhost:7878` · Prowlarr `localhost:9696` · qBittorrent `localhost:8080`.

Para que el navegador confíe en el certificado local:
```bash
docker compose cp caddy:/data/caddy/pki/authorities/local/root.crt ./caddy-root.crt
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain caddy-root.crt
```
Reinicia el navegador (Cmd+Q). Si Safari no encuentra `watch.localhost`, agrega esta línea a `/etc/hosts`:
```text
127.0.0.1 watch.localhost request.localhost
```

## 6. Siguiente paso

Configura las apps, el acceso remoto con tu dominio y los respaldos en **[configuracion.md](configuracion.md)**.
