# Despliegue en Windows (WSL2)

Cómo levantar el stack en Windows 10/11 con Docker Desktop y WSL2. Al terminar, sigue con
[configuracion.md](configuracion.md): apps, acceso remoto y mantenimiento.

Todos los valores de ejemplo (`example.com`, `192.168.1.50`, claves `<...>`) son placeholders: los reales
van solo en tu `.env`, que no se sube al repo.

---

## 1. Preparar Windows

1. **WSL2 + Ubuntu** (PowerShell como administrador):
   ```powershell
   wsl --install -d Ubuntu
   ```
   Reinicia y crea tu usuario de Ubuntu cuando lo pida.
2. **Docker Desktop**: instálalo y en *Settings* activa:
   - *General → Use the WSL 2 based engine*.
   - *General → Start Docker Desktop when you sign in*.
   - *Resources → WSL integration → Ubuntu*.
3. **Que no se duerma** (PowerShell como administrador): tapa cerrada sin suspender, con corriente y con batería.
   ```powershell
   powercfg /setacvalueindex SCHEME_CURRENT SUB_BUTTONS LIDACTION 0
   powercfg /setdcvalueindex SCHEME_CURRENT SUB_BUTTONS LIDACTION 0
   powercfg /change standby-timeout-ac 0
   powercfg /setactive SCHEME_CURRENT
   ```
   En *Windows Update → Opciones avanzadas → Horas activas*, cubre tu horario para evitar reinicios a media película.
4. **IP fija en la LAN**: reserva una IP para la laptop en el DHCP del router (ej. `192.168.1.50`) y usa cable.
5. **Red como "Privada"**: *Configuración → Red → Ethernet → Tipo de perfil: Privada*. El firewall de Windows
   solo abrirá los paneles de admin a la LAN en ese perfil.

## 2. Clonar y configurar (dentro de WSL)

Abre **Ubuntu** (no PowerShell) y clona en el home de Linux, **no** en `/mnt/c`: ahí es lento y puede
romper permisos.

```bash
cd ~ && git clone https://github.com/Molisawa/jellyfin-selfhosted-liquid-media.git && cd jellyfin-selfhosted-liquid-media
cp .env.example .env
```

Valores de producción en `.env`:

| Variable | Prod | Por qué |
|---|---|---|
| `PUID` / `PGID` | salida de `id -u` / `id -g` (normalmente `1000`) | dueño de los archivos de media |
| `TZ` | ej. `America/Mexico_City` | horarios de tareas y logs |
| `MEDIA_PATH` | ver §3 | raíz única para hardlinks |
| `DOMAIN` | `example.com` (el tuyo) | Caddy pide certificados Let's Encrypt para `watch.` y `request.` |
| `ACME_EMAIL` | tu correo real | Let's Encrypt rechaza `example.com` |
| `LAN_BIND_IP` | `0.0.0.0` | paneles admin visibles en la LAN (no en internet) |
| `COMPOSE_PROFILES` | `novpn` o `vpn` | qBittorrent directo o detrás de Gluetun |

> Los volúmenes nombrados (`/config` de cada app) viven dentro de la VM de Docker, en ext4. Así SQLite no
> toca NTFS. **No** cambies esos volúmenes a rutas `C:\...`.

## 3. Almacenamiento y hardlinks

Los hardlinks solo funcionan si `torrents/` y `media/` están en **el mismo sistema de archivos**. Por eso
todo cuelga de un único `MEDIA_PATH`. Hay dos opciones:

### Opción A — Disco NTFS de Windows (simple)

```bash
mkdir -p /mnt/d/MediaServer/data/{torrents/{movies,tv},media/{movies,tv}}
```
En `.env`: `MEDIA_PATH=/mnt/d/MediaServer/data`. Ventaja: ves y copias tus archivos desde el Explorador.
Desventaja: el acceso a NTFS desde contenedores pasa por una capa de traducción, más lenta, y **los
hardlinks hay que comprobarlos** (abajo).

### Opción B — Disco dedicado en ext4 (más robusto)

Si tienes un disco solo para media (borra su contenido). Requiere Windows 11:
```powershell
# PowerShell admin: identifica el número del disco (columna Number) y móntalo en WSL
Get-Disk
wsl --mount \\.\PHYSICALDRIVE1 --bare
```
```bash
# En Ubuntu: formatea UNA vez (sdX = el disco nuevo, revisa con lsblk) y monta
lsblk
sudo mkfs.ext4 /dev/sdX
sudo mkdir -p /srv/media && sudo mount /dev/sdX /srv/media && sudo chown -R $(id -u):$(id -g) /srv/media
mkdir -p /srv/media/data/{torrents/{movies,tv},media/{movies,tv}}
```
En `.env`: `MEDIA_PATH=/srv/media/data`. Rendimiento y hardlinks nativos. Desventaja: desde el Explorador se
ve solo vía `\\wsl$\Ubuntu\srv\media`, y `wsl --mount` hay que repetirlo tras cada reinicio (una tarea
programada de Windows al iniciar sesión lo resuelve).

### Comprobar hardlinks (haz esto antes de configurar Sonarr/Radarr)

```bash
docker compose up -d sonarr
docker compose exec sonarr sh -c 'echo test > /data/torrents/tv/.hl && ln /data/torrents/tv/.hl /data/media/tv/.hl && stat -c "%h enlaces" /data/media/tv/.hl; rm -f /data/torrents/tv/.hl /data/media/tv/.hl'
```
Debe imprimir **`2 enlaces`**. Si falla o imprime `1`, usa la opción B.

## 4. Levantar el stack

Sin GPU:
```bash
docker compose up -d
```
Con GPU **NVIDIA**: instala el driver normal de NVIDIA **en Windows**, no dentro de WSL.
```bash
docker compose -f docker-compose.yml -f docker-compose.nvidia.yml up -d
docker compose exec jellyfin nvidia-smi
```
Si `nvidia-smi` muestra la tarjeta, en Jellyfin ve a *Panel → Reproducción → Transcodificación → NVIDIA NVENC*.

> **Aviso:** la documentación de Jellyfin indica que, con Docker Desktop, la GPU NVIDIA a veces se pierde hasta
> reiniciar el contenedor. Si te pasa seguido, la alternativa es Docker Engine instalado dentro de WSL con
> `nvidia-container-toolkit`; solo vale la pena si el problema se repite, porque complica la red. Intel
> QuickSync (`/dev/dri`) **no** está disponible en WSL2; `docker-compose.intel.yml` es solo para Linux nativo.

## 5. Siguiente paso

Configura las apps, el acceso remoto con tu dominio y los respaldos en **[configuracion.md](configuracion.md)**.

