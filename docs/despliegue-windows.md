# Despliegue en Windows (WSL2) y acceso remoto

Guía para llevar el stack de la Mac (dev) a una laptop con Windows (prod). Todos los valores de ejemplo
(`example.com`, `192.168.1.50`, `203.0.113.10`, claves `<...>`) son placeholders: cámbialos solo en tu `.env`
o en archivos que **no** se suben al repo.

**Qué cambia respecto a la Mac:** el código, el `docker-compose.yml` y el `Caddyfile` son los mismos.
Solo cambian `.env`, dónde vive el disco de media y la red (router/DNS).

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
> `nvidia-container-toolkit`, aunque complica la red; pregunta antes de migrar. Intel QuickSync (`/dev/dri`)
> **no** está disponible en WSL2; `docker-compose.intel.yml` es solo para Linux nativo.

## 5. Configurar las apps (en este orden)

Todas se hablan por nombre de contenedor dentro de `media_net`. **Nunca** uses `localhost` entre apps.

1. **qBittorrent** — `http://<IP-LAN>:8080`. Usuario `admin`; la contraseña temporal sale con
   `docker compose logs qbittorrent | grep -i password`.
   - *Opciones → WebUI*: pon tu propia contraseña.
   - *Opciones → Descargas*: ruta por defecto `/data/torrents`.
   - Categorías: `movies` → `/data/torrents/movies`, `tv` → `/data/torrents/tv`.
2. **Prowlarr** — `http://<IP-LAN>:9696`. *Settings → Apps*: agrega Sonarr (`http://sonarr:8989`) y Radarr
   (`http://radarr:7878`), con Prowlarr server `http://prowlarr:9696` y la API key de cada app
   (*Settings → General* en cada una). Los indexers dependen de ti; este repo no incluye ninguno.
3. **Sonarr** (`:8989`) y **Radarr** (`:7878`):
   - *Settings → Media Management*: activa **Use Hardlinks instead of Copy** (viene activado).
     Carpeta raíz: `/data/media/tv` (Sonarr) o `/data/media/movies` (Radarr).
   - *Settings → Download Clients → qBittorrent*: host `qbittorrent`, puerto `8080`, categoría `tv` o `movies`.
   - *Remote Path Mappings*: **ninguno**. Las rutas son idénticas en todos los contenedores.
4. **Jellyfin** — `https://watch.<tu-dominio>` o `http://<IP-LAN>:8096`.
   - Bibliotecas: `/data/media/movies` y `/data/media/tv`.
   - *Panel → Redes*: **Proxies conocidos** = `caddy`; *Permitir conexiones remotas* activado.
   - Tema: *Panel → General → CSS personalizado*: `@import url("/custom-theme/liquid-glass.css");`
5. **Seerr** — `https://request.<tu-dominio>`. Conéctalo a Jellyfin en `http://jellyfin:8096`, y a Sonarr y
   Radarr por sus nombres internos.

Cuando Sonarr o Radarr importen algo, confirma que fue hardlink: el archivo en `media/` debe mostrar
**2 enlaces**.
```bash
docker compose exec radarr sh -c 'find /data/media -type f -name "*.mkv" -exec stat -c "%h %n" {} + | head'
```

## 6. Acceso remoto

Solo `watch.` (Jellyfin) y `request.` (Seerr) salen a internet, siempre a través de Caddy en 80/443.

**¿Tienes CGNAT?** Compara la IP WAN que muestra tu router con la de `curl -s https://ifconfig.me`. Si
**no** coinciden, o la del router está en `100.64.0.0/10`, `10.x`, `172.16–31.x` o `192.168.x`, estás detrás
de CGNAT y aplica el **Caso B**.

### DNS (ambos casos)

Crea dos registros **A**, `watch` y `request`, apuntando a tu IP pública (Caso A) o a la del VPS (Caso B).
Con **Cloudflare**, `watch` debe ir en **Solo DNS (nube gris)**: su CDN no permite servir video y además
rompería el streaming. `request` puede ir igual.

### Caso A — IP pública (sin CGNAT)

1. En el router, *Port forwarding* a la IP LAN de la laptop: **TCP 80**, **TCP 443** y **UDP 443** (HTTP/3).
2. Si tu IP pública cambia, activa **DDNS** (casi todos los routers lo traen) o un cliente DDNS de tu
   proveedor DNS que actualice los dos registros.
3. Reinicia Caddy para que pida certificados:
   ```bash
   docker compose restart caddy && docker compose logs -f caddy
   ```
   Busca `certificate obtained successfully` para `watch.` y `request.`.

### Caso B — Detrás de CGNAT (túnel WireGuard a un VPS)

El VPS (cualquiera barato con IP pública) solo **reenvía TCP/UDP**. El HTTPS sigue terminando en tu Caddy,
así que el VPS nunca ve el contenido descifrado ni los certificados.

```
Internet ──80/443──▶ VPS (203.0.113.10) ══WireGuard══▶ Laptop Windows (10.8.0.2) ──▶ Caddy ──▶ Jellyfin / Seerr
```

**VPS (Ubuntu)**:
```bash
sudo apt install -y wireguard
wg genkey | tee vps.key | wg pubkey > vps.pub
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-forward.conf && sudo sysctl --system
```
`/etc/wireguard/wg0.conf` (cambia `eth0` por la interfaz pública de tu VPS, que ves con `ip route`):
```ini
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <contenido de vps.key>
PostUp   = iptables -t nat -A PREROUTING -i eth0 -p tcp -m multiport --dports 80,443 -j DNAT --to-destination 10.8.0.2
PostUp   = iptables -t nat -A PREROUTING -i eth0 -p udp --dport 443 -j DNAT --to-destination 10.8.0.2
PostUp   = iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
PostDown = iptables -t nat -D PREROUTING -i eth0 -p tcp -m multiport --dports 80,443 -j DNAT --to-destination 10.8.0.2
PostDown = iptables -t nat -D PREROUTING -i eth0 -p udp --dport 443 -j DNAT --to-destination 10.8.0.2
PostDown = iptables -t nat -D POSTROUTING -o wg0 -j MASQUERADE

[Peer]
PublicKey = <clave pública de la laptop>
AllowedIPs = 10.8.0.2/32
```
```bash
sudo systemctl enable --now wg-quick@wg0
```
Abre en el firewall del VPS **UDP 51820**, **TCP 80/443** y **UDP 443**. Con `ufw`, también hay que permitir el
reenvío: `DEFAULT_FORWARD_POLICY="ACCEPT"` en `/etc/default/ufw`.

**Laptop Windows**: instala el cliente oficial de WireGuard para Windows y crea un túnel vacío (te genera
la clave; copia su clave pública al `[Peer]` del VPS):
```ini
[Interface]
PrivateKey = <generada por la app>
Address = 10.8.0.2/32

[Peer]
PublicKey = <contenido de vps.pub>
Endpoint = 203.0.113.10:51820
AllowedIPs = 10.8.0.1/32
PersistentKeepalive = 25
```
`AllowedIPs` solo incluye al VPS, así que el resto de tu tráfico **no** pasa por el túnel. Activa el túnel
y márcalo para que arranque con Windows. Si el firewall de Windows pregunta por Docker Desktop, permite
redes **públicas y privadas**: Windows clasifica la interfaz de WireGuard como pública.

Prueba: `curl -I https://watch.example.com` desde tu celular **con datos móviles** debe responder `302`.

> **Alternativa sin abrir nada:** si solo tu familia va a usar Jellyfin, un VPN privado tipo Tailscale en cada
> dispositivo evita exponer el servidor. Las apps de TV a veces no lo soportan; por eso existe el Caso B.

### Detalles a saber

- **Desde tu casa**, `watch.example.com` puede no cargar si el router no soporta *NAT loopback*. Usa
  `http://<IP-LAN>:8096` en la LAN, o una entrada DNS local en el router.
- **IP real de los clientes:** Docker Desktop (y el túnel del Caso B) reescriben la IP de origen, así que
  Jellyfin ve una IP interna para todos. Funciona igual, pero los límites de bitrate "remoto" y el registro
  de IPs no distinguen clientes.

## 7. Checklist de seguridad

- [ ] El router solo reenvía 80/443 (o nada, en el Caso B). Nunca 8096, 8989, 7878, 9696 ni 8080.
- [ ] Contraseñas propias y fuertes en Jellyfin, Seerr y qBittorrent. Sonarr, Radarr y Prowlarr con
      *Authentication: Forms* en *Settings → General*.
- [ ] Nada de los datos de dev: en prod empiezas con volúmenes nuevos (sin `devadmin`).
- [ ] `.env` nunca se sube (`git status` no debe listarlo).
- [ ] Perfil de red de Windows en **Privada**.

## 8. Mantenimiento

Actualizar (las apps guardan su config en los volúmenes):
```bash
git pull && docker compose pull && docker compose up -d && docker image prune -f
```
Respaldo de la config de una app (ejemplo: Jellyfin), apagada para no copiar una base SQLite a medias:
```bash
docker compose stop jellyfin
docker run --rm -v jellyfin-selfhosted-liquid-media_jellyfin_config:/v -v "$PWD":/b alpine tar czf /b/jellyfin-config.tgz -C /v .
docker compose start jellyfin
```
Los `.tgz` contienen tokens y usuarios: guárdalos fuera del repo.
