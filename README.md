# home-lab

Mi *homelab* self-hosted corriendo sobre una **Raspberry Pi 5 (ARM64)** con Docker Compose:
un stack de *media*, *arr apps*, descargas y servicios de red, todo detrás de un reverse
proxy nginx con TLS de Let's Encrypt.

> ⚠️ Este repo contiene **solo configuración**. No incluye secretos: todos los valores
> sensibles se inyectan mediante variables de entorno desde un `.env` (que está en
> `.gitignore`). El dominio y las IPs de la LAN están anonimizados (`example.com`,
> `192.168.1.x`) — ajústalos a los tuyos.

## Contenido

```
docker-compose.yml     # definición de todos los servicios
nginx/conf.d/          # reverse proxy: vhosts *.local (LAN) y *.example.com (público con TLS)
.env.example           # plantilla de variables — copiar a .env
```

## Servicios (resumen)

- **Media:** Jellyfin, Navidrome, Calibre-Web, MeTube, Pixelfin
- **Gestión (*arr*):** Prowlarr, Radarr, Sonarr, Lidarr, Bazarr, Whisparr, Readarr/Bookshelf, Profilarr, Cleanuparr, Maintainerr, Jellyseerr, Recommendarr
- **Descargas:** ruTorrent + Flood, SABnzbd, aMule (+ amulerr)
- **Red / infra:** nginx, certbot, Pi-hole (DNS), Cloudflare DDNS, Portainer, Watchtower, Syncthing, Samba, dashdot, OpenSpeedTest
- **Analítica:** Streamystats (+ Postgres)

## Uso

```bash
git clone <este-repo>
cd home-lab
cp .env.example .env
# edita .env con tus secretos, dominio e IPs reales
docker compose up -d
```

### Notas de la plataforma (ARM64)

La Pi es **arm64 y sin QEMU**. Antes de desplegar una imagen, verifica que publique
manifiesto `linux/arm64`; si es amd64-only, hay que compilarla localmente
(ej. `questarr:local`, ver comentario en el `docker-compose.yml`).

### Reverse proxy

- `*.local` → acceso interno en la LAN (mapeo host→servicio en `nginx/conf.d/00-maps.conf`).
- `*.example.com` → acceso público con TLS (certificados gestionados por certbot en `41-*`, `42-*`, …).

## Licencia

MIT
