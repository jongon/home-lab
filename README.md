# home-lab

Mi *homelab* self-hosted sobre una **Raspberry Pi 5 (ARM64)** orquestado con Docker
Compose: servidor multimedia, automatización de descargas (*\*arr apps*), servicios de
red y un reverse proxy **nginx** con TLS de **Let's Encrypt**. Acceso interno por
`*.local` y publicación selectiva de algunos servicios en Internet con HTTPS.

> ⚠️ Este repo contiene **solo configuración**, sin secretos. Todos los valores
> sensibles se inyectan como variables de entorno desde un `.env` (ignorado por git;
> usa `.env.example` como plantilla). El dominio real y las IPs de la LAN están
> anonimizados (`example.com`, `192.168.1.x`): sustitúyelos por los tuyos.

---

## Índice

- [Arquitectura](#arquitectura)
- [Servicios de terceros que hay que habilitar](#servicios-de-terceros-que-hay-que-habilitar)
- [Catálogo de servicios](#catálogo-de-servicios)
- [Cómo funciona el reverse proxy (nginx)](#cómo-funciona-el-reverse-proxy-nginx)
- [Puesta en marcha](#puesta-en-marcha)
- [Notas de la plataforma (ARM64)](#notas-de-la-plataforma-arm64)

---

## Arquitectura

```
                    Internet
                       │
             Cloudflare DNS (registros A)  ◄── cloudflare-ddns actualiza la IP pública
                       │
             Router (port-forward 80/443 + puertos P2P)
                       │
        ┌──────────────┴───────────────┐
        │        Raspberry Pi 5         │
        │                               │
        │   nginx  ──►  certbot (TLS)   │
        │     │                         │
        │     ├─ *.example.com  (443, público, HTTPS)
        │     └─ *.local        (80, solo LAN, vía Pi-hole DNS)
        │             │                 │
        │   docker network «backend» (172.19.0.0/16)
        │     ├─ Jellyfin, Navidrome, Calibre, …
        │     ├─ Prowlarr/Radarr/Sonarr/… (*arr)
        │     ├─ ruTorrent, SABnzbd, aMule (descargas)
        │     └─ Pi-hole (DNS 172.19.0.53) ◄── DNS de todos los contenedores
        └───────────────────────────────┘
```

**Almacenamiento** (montado en el host):
- `/mnt/nvme/<servicio>` → configuración y datos persistentes de cada contenedor (SSD NVMe).
- `/media/media` → biblioteca multimedia (Movies, Tv Series, Music, Books, Youtube, …).
- `/media/downloads` → carpeta de descargas compartida entre clientes y *arr apps*.

**Redes Docker**:
- `backend` (`172.19.0.0/16`): red principal. **Pi-hole tiene IP fija `172.19.0.53`** y
  es el **servidor DNS de todos los contenedores** (ancla `x-dns`), con `1.1.1.1` de
  respaldo. Así el tráfico de los contenedores también se filtra por Pi-hole.
- `openclaw_net`: red aislada para el servicio `openclaw` (solo comparte con nginx).

**Identidad y zona horaria**: `PUID`/`PGID` (1000) y `TZ` se comparten vía el ancla
YAML `x-common-env` para que todos los contenedores escriban ficheros con el mismo
usuario y usen la misma hora local.

---

## Servicios de terceros que hay que habilitar

Para que **todo** el stack funcione de extremo a extremo necesitas configurar estos
servicios/recursos externos (no son contenedores, son dependencias de fuera):

| Servicio externo | Para qué | Qué hay que hacer |
|---|---|---|
| **Cloudflare** (dominio) | DNS público + IP dinámica | Tener el dominio con DNS gestionado por Cloudflare. Crear un **API Token** con permiso `Zone → DNS → Edit` sobre tu zona → va en `CLOUDFLARE_TOKEN`. Crear registros **A** para cada subdominio público (`jellyfin`, `ha`, `navidrome`, `jellyseerr`, …) apuntando a tu IP pública. El contenedor `cloudflare-ddns` mantiene la IP actualizada. Se usa `PROXIED=false` (DNS-only) para que el streaming y la validación ACME lleguen directos a tu servidor. |
| **Router / ISP** | Exponer los servicios públicos | **Port-forward** de `80/tcp` y `443/tcp` → IP de la Pi (necesario para Let's Encrypt y el acceso HTTPS). Para P2P: reenviar `51413/tcp` y `6881/udp` (rTorrent) y `4662/tcp`, `4672/udp`, `4665/udp` (aMule). IP pública preferiblemente estática o, si es dinámica, cubierta por el DDNS de Cloudflare. |
| **Let's Encrypt** | Certificados TLS gratuitos | Los emite el contenedor `certbot` por método *webroot* (`.well-known/acme-challenge`). Requiere que el puerto 80 sea accesible desde Internet y que el DNS del subdominio ya resuelva a tu IP. La **primera emisión es manual** (ver [Puesta en marcha](#puesta-en-marcha)); `certbot-renew` se encarga de renovar cada 12 h. |
| **DNS local para `*.local`** | Acceso interno por nombre | Los dominios `*.local` (p. ej. `radarr.local`) deben resolver a la IP de la Pi. Configúralo en **Pi-hole** (Local DNS / wildcard) o en tu router, y haz que los clientes de la LAN usen Pi-hole como DNS. Sin esto, el acceso interno por nombre no funciona (el público por `*.example.com` sí, porque resuelve por Cloudflare). |
| **OpenAI API** (opcional) | Funciones de IA | `recommendarr` y `digarr` usan un proveedor de IA. Necesitas una API key en `AI_API_KEY` (y ajustar `AI_PROVIDER`/`AI_MODEL`). Puedes sustituirlo por otro proveedor compatible. |
| **Indexers / trackers** | Fuentes de contenido | Se configuran **dentro de Prowlarr** (y este los distribuye al resto de *arr*). Algunos indexers privados requieren cuenta/API key propia. |
| **Hardcover** (opcional) | Metadatos de libros | La imagen `bookshelf:hardcover` se integra con [Hardcover](https://hardcover.app); requiere token si usas esa integración. |

---

## Catálogo de servicios

### 📺 Multimedia
| Servicio | Puerto | Función |
|---|---|---|
| **Jellyfin** | 8096 | Servidor multimedia principal (películas, series, música, libros). Publicado en HTTPS. |
| **Navidrome** | 4533 | Servidor de música con API Subsonic (apps móviles de streaming). Publicado en HTTPS. |
| **Calibre-Web** | 8083 | Biblioteca y lector web de ebooks. |
| **MeTube** | 8081 | Front-end web de `yt-dlp` para descargar vídeo/audio (p. ej. de YouTube) a la biblioteca. |
| **Pixelfin** | 1280 | Gestión/generación de carátulas y artwork para Jellyfin. |
| **Meilisearch** | 7700 | Motor de búsqueda rápido usado como backend por apps compañeras. |

### 🎬 Gestión de biblioteca (*\*arr* stack)
| Servicio | Puerto | Función |
|---|---|---|
| **Prowlarr** | 9696 | Gestor central de *indexers*; sincroniza las fuentes al resto de *arr*. |
| **Radarr** | 7878 | Gestión y descarga automática de **películas**. |
| **Sonarr** | 8989 | Gestión y descarga automática de **series de TV**. |
| **Lidarr** | 8686 | Gestión y descarga automática de **música**. |
| **Bazarr** | 6767 | Descarga automática de **subtítulos** para Radarr/Sonarr. |
| **Whisparr** | 6969 | *arr* para contenido **adulto** (XXX). |
| **Readarr / Bookshelf** | 8787 | Gestión de **libros/audiolibros** (integración Hardcover). |
| **Lingarr** | 9876 | **Traducción** automática de subtítulos. |
| **Profilarr** (+ parser) | 6868 | Sincroniza perfiles de calidad y *custom formats* entre las *arr apps*. |
| **Cleanuparr** | 11011 | Limpieza de descargas bloqueadas/huérfanas en los clientes. |
| **Maintainerr** | 6246 | Reglas automáticas para retirar contenido antiguo/no visto de la biblioteca. |
| **Jellyseerr** | 5055 | Portal de **peticiones** de contenido para usuarios de Jellyfin. Publicado en HTTPS. |
| **Recommendarr** | 3000 | **Recomendaciones** con IA cruzando Jellyfin + *arr* stack. |
| **digarr** | 3000 | Compañero de Lidarr con IA (descubrimiento/automatización musical). |
| **muxarr** | 8183 | Automatización de *remux/muxing* de la biblioteca. |
| **Questarr** | 5000 | Gestor estilo *arr* para **videojuegos** (enruta descargas a la biblioteca de juegos). |

### ⬇️ Clientes de descarga
| Servicio | Puerto | Función |
|---|---|---|
| **ruTorrent** (rTorrent) | 8080 | Cliente **BitTorrent** con WebUI clásica. Puertos P2P 51413/6881. |
| **Flood** | 3001 | WebUI moderna alternativa para el mismo rTorrent. |
| **SABnzbd** | 8080 | Cliente **Usenet** (NZB). |
| **aMule** (+ **amulerr**) | 4711 / — | Cliente **eD2k/Kademlia**; `amulerr` da integración estilo *arr*. |

### 🌐 Red e infraestructura
| Servicio | Puerto | Función |
|---|---|---|
| **nginx** | 80/443 | **Reverse proxy** y terminación TLS (ver sección dedicada). |
| **certbot** / **certbot-renew** | — | Emisión y **renovación** automática de certificados Let's Encrypt (webroot). |
| **Pi-hole** | 53/80 | **DNS** con bloqueo de anuncios para toda la red y DNS interno de los contenedores. |
| **Cloudflare-DDNS** | — | Mantiene actualizada la **IP pública** en los registros de Cloudflare. |
| **Portainer** | 9000 | UI de gestión de Docker. |
| **Watchtower** | — | **Auto-actualización** de contenedores (cada día a las 10:00). |
| **Syncthing** | 8384 | Sincronización de ficheros P2P entre dispositivos. |
| **Samba** | 139/445 | Compartición de ficheros **SMB/CIFS** de `/media` en la LAN. |
| **dashdot** | 3001 | Dashboard de monitorización del servidor (CPU, disco, red, temperatura). |
| **OpenSpeedTest** | 3000 | Test de velocidad de red autoalojado. |

### 📊 Analítica
| Servicio | Puerto | Función |
|---|---|---|
| **Streamystats** (+ job + Postgres) | 3000 | Panel de **estadísticas de visionado** de Jellyfin. Usa Postgres (`vchord`) propio. |

### 🛠️ Herramientas CLI (perfil `cli`, no arrancan por defecto)
| Servicio | Función |
|---|---|
| **managarr** | TUI para administrar las *arr apps* desde terminal. |
| **openclaw** / **openclaw-cli** | Gateway/agente autoalojado enlazado a la LAN (red aislada `openclaw_net`). |
| **opencode** | Agente de codificación por IA en terminal. |

> Los servicios con `profiles: ["cli"]` solo arrancan si los invocas explícitamente
> (`docker compose run --rm <servicio>`), no con `docker compose up -d`. Es el patrón
> que uso para herramientas interactivas de un solo uso.

---

## Cómo funciona el reverse proxy (nginx)

Toda la configuración vive en `nginx/conf.d/`, que se monta en el contenedor. nginx
carga los ficheros **en orden alfabético**, por eso el prefijo numérico. El diseño
tiene **dos capas de acceso**:

### 1. Acceso interno por `*.local` (LAN, HTTP)
En vez de un `server {}` por cada app, hay **un único vhost genérico** que resuelve el
destino de forma dinámica:

- **`00-maps.conf`** define dos `map`: `$host → $upstream_svc` (nombre del contenedor)
  y `$host → $upstream_port`. Añadir un servicio nuevo = añadir dos líneas aquí.
- **`10-local-generic.conf`** captura cualquier `*.local` con la regex
  `~^(?<name>.+)\.local$` y hace `proxy_pass http://$upstream_svc:$upstream_port`. Si
  el host no está en el mapa, devuelve `444` (cierra la conexión).
- **`00-resolver.conf`** apunta al DNS interno de Docker (`127.0.0.11`). Es
  **imprescindible** porque, al usar variables en `proxy_pass`, nginx resuelve el
  nombre del contenedor en *runtime* y no al arrancar.

👉 Requiere que el DNS de la LAN (Pi-hole) resuelva `*.local` a la IP de la Pi.

### 2. Acceso público por `*.example.com` (Internet, HTTPS)
Cada servicio expuesto tiene su propio fichero (`41-ha.conf`, `42-jellyfin.conf`,
`43-navidrome.conf`, `44-jellyseerr.conf`) escuchando en `443 ssl` con su certificado:

- **`20-public-redirect.conf`** escucha en `:80`, sirve el reto ACME
  (`/.well-known/acme-challenge/`) y redirige todo lo demás a HTTPS (`301`).
- **`30-security-headers.inc`** es un *include* con cabeceras de seguridad (HSTS,
  `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`, …) reutilizado por
  todos los vhosts públicos.
- **`00-connection.conf`** define el `map` de `Upgrade` para soportar **WebSockets**
  (necesario para Jellyfin, Home Assistant, etc.).

### 3. Certificados TLS (flujo con certbot)
- `certbot` y `nginx` comparten dos volúmenes: `certbot/www` (webroot del reto ACME) y
  `certbot/conf` (`/etc/letsencrypt`, donde viven los certs).
- `certbot-renew` ejecuta `certbot renew` cada 12 h y, tras renovar, hace `touch` de un
  fichero *flag* (`.nginx-reload`). Un bucle dentro del contenedor nginx detecta ese
  flag y ejecuta `nginx -s reload` **sin reiniciar** el contenedor.

### 4. Proxy a otras máquinas de la LAN
No todo apunta a contenedores: algunos vhosts hacen proxy a otros equipos por IP:
- **`40-ha-local.conf`** / **`41-ha.conf`** → Home Assistant (`192.168.1.10`), en local y público.
- **`45-mikrotik.conf`** → interfaz del router MikroTik (`192.168.1.1`).
- **`46-bazzite-syncthing.conf`** → Syncthing de otra máquina (`192.168.1.20`).
- **`50-opentest.conf`** → sirve OpenSpeedTest en el puerto 3000.

---

## Puesta en marcha

```bash
git clone https://github.com/jongon/home-lab.git
cd home-lab
cp .env.example .env
#   edita .env con tus secretos, dominio e IPs reales
#   ajusta nginx/conf.d/*.conf (dominio example.com → el tuyo, IPs de la LAN)
```

1. **Prepara los prerequisitos externos** de la tabla de arriba (Cloudflare, port-forward, DNS local).
2. **Crea las carpetas de datos** en `/mnt/nvme` y `/media` con el `PUID:PGID` correcto.
3. **Emite los certificados por primera vez** (con nginx sirviendo el puerto 80):
   ```bash
   docker compose up -d nginx certbot
   docker compose run --rm certbot certonly --webroot -w /var/www/certbot \
     -d jellyfin.example.com -d ha.example.com -d navidrome.example.com -d jellyseerr.example.com
   ```
4. **Levanta el resto del stack**:
   ```bash
   docker compose up -d
   ```
5. Configura cada app desde su UI (Prowlarr → indexers, *arr* → rutas de biblioteca y clientes de descarga, etc.).

---

## Notas de la plataforma (ARM64)

La Raspberry Pi es **arm64 y sin QEMU**. Antes de desplegar una imagen, comprueba que
publique manifiesto `linux/arm64`; si es *amd64-only*, hay que **compilarla localmente**
(ejemplo: `questarr:local`, ver el comentario en el `docker-compose.yml`).

**Watchtower** actualiza automáticamente las imágenes cada día; si prefieres control
manual, desactívalo o usa etiquetas de exclusión.

## Licencia

MIT — ver [LICENSE](LICENSE).
