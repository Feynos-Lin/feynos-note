# Feynos Notes

OneNote-style notes app for **Android**, **Fedora (.rpm)** and **Windows**, plus a self-hosted **sync server**,
all built from [`.github/workflows/feynos-notes.yml`](.github/workflows/feynos-notes.yml). Every push to `main`
builds everything and publishes it under **Releases**; the server image goes to `ghcr.io/feynos-lin/feynos-note-sync`.

- Notebooks → colored sections → pages and subpages
- Rich text: headings, colors, highlight, lists, to-do checkboxes, tags, tables, links, pictures, sketches
- Search across every page, recycle bin, light/dark theme
- **Live sync** between all your devices through your own server (works behind a reverse proxy)
- **Import OneNote notebooks from OneDrive**
- **Backups to Google Drive & Proton Drive** (desktop: sync folders; Android: share sheet) and restore from file

---

## Sync server

1. Copy [`docker-compose.yml`](#docker-composeyml) (also attached to every release) into a folder on your server.
2. Set `FEYNOS_PASSWORD` to something long.
3. `docker compose up -d`
4. Open `http://SERVER-IP:8787` in a browser — you should see *“Feynos Notes sync server is running”*.
5. In the app: **Settings → Sync**, enter the address and password, and connect. Do this on each device.

The first device to connect uploads its notes. A device with only the starter notebook just downloads what's on the server.
If two devices edit the same page before syncing, both versions are kept (the second shows up as a *conflict copy*).

### Settings

| Variable | Default | What it does |
|---|---|---|
| `FEYNOS_PASSWORD` | — (required) | Password the apps sign in with. 8+ characters. Or use `FEYNOS_PASSWORD_FILE` for a Docker secret. |
| `TRUST_PROXY` | `false` | Use `X-Forwarded-For` / `X-Real-IP` from your proxy for logs and login rate limiting. Set `true` behind a proxy. |
| `BASE_PATH` | *(none)* | Serve under a sub-path, e.g. `/notes` for `https://example.com/notes`. Works whether or not the proxy strips the prefix. |
| `MAX_BODY_MB` | `200` | Largest single upload. Keep your proxy's limit at least this high. |
| `CORS_ORIGINS` | `*` | Allowed browser origins. Sign-in uses tokens, not cookies, so `*` is safe. |
| `MS_CLIENT_ID` | *(none)* | Enables **Import from OneNote** — see below. |
| `MS_TENANT` | `common` | Use `consumers` for personal Microsoft accounts only. |
| `PORT` | `8787` | Port inside the container. |

Data lives in `/data` (one SQLite file). Back up that folder, or keep using the in-app Google Drive / Proton Drive backups.

## Reverse proxy

Point your proxy at `http://<docker-host>:8787` and set `TRUST_PROXY=true`. Three things matter:

- **Upload size:** pages with pictures can be large. Allow at least `MAX_BODY_MB` (200 MB).
- **Timeouts:** the first sync or a big OneNote import can take a few minutes. Allow 300 s+.
- **HTTPS:** use a real certificate (Let's Encrypt). Android rejects self-signed certificates.

**Nginx Proxy Manager** — add a Proxy Host: forward to the server IP, port `8787`, scheme `http`. On the SSL tab request a Let's Encrypt certificate and enable *Force SSL*. On the Advanced tab add:

```nginx
client_max_body_size 200m;
proxy_read_timeout 300s;
proxy_send_timeout 300s;
```

**Caddy**

```caddy
notes.example.com {
    request_body {
        max_size 200MB
    }
    reverse_proxy feynos-notes-sync:8787
}

# or on a sub-path (set BASE_PATH=/notes)
example.com {
    handle /notes* {
        reverse_proxy feynos-notes-sync:8787
    }
}
```

**Nginx**

```nginx
server {
    listen 443 ssl;
    server_name notes.example.com;
    # ssl_certificate ...; ssl_certificate_key ...;

    client_max_body_size 200m;
    proxy_read_timeout 300s;
    proxy_send_timeout 300s;

    location / {
        proxy_pass http://127.0.0.1:8787;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

For a sub-path use `location /notes/ { proxy_pass http://127.0.0.1:8787; ... }` with `BASE_PATH=/notes`.

**Traefik** (labels on the service in `docker-compose.yml`)

```yaml
    labels:
      - traefik.enable=true
      - traefik.http.routers.feynos-notes.rule=Host(`notes.example.com`)
      - traefik.http.routers.feynos-notes.entrypoints=websecure
      - traefik.http.routers.feynos-notes.tls.certresolver=letsencrypt
      - traefik.http.services.feynos-notes.loadbalancer.server.port=8787
```

Traefik has no upload limit by default. If you add the `buffering` middleware, set `maxRequestBodyBytes` to 200 MB or more.

**Tailscale / LAN only:** skip the proxy and use `http://SERVER-IP:8787` (or the Tailscale name) directly. The apps allow plain HTTP for this.

## OneNote import setup

The import runs on your sync server and signs in to Microsoft with a code, so it works the same from every device.
It needs a free Microsoft app registration (one time, about 5 minutes):

1. Go to <https://entra.microsoft.com> → **Applications → App registrations → New registration**.
   (If your personal account has no directory yet, Microsoft may ask you to create a free Azure account first.)
2. Name it `Feynos Notes import`. Under *Supported account types* choose **Personal Microsoft accounts only**
   (or *any organizational directory and personal accounts* if you also use a work/school account). Leave Redirect URI empty. **Register**.
3. **Authentication** → *Advanced settings* → **Allow public client flows: Yes** → Save.
4. **API permissions** → *Add a permission* → Microsoft Graph → **Delegated** → `Notes.Read` and `User.Read` → Add.
5. Copy the **Application (client) ID** from *Overview* into `MS_CLIENT_ID` in `docker-compose.yml`
   (with personal accounts only, also set `MS_TENANT=consumers`), then `docker compose up -d`.

In the app: **Settings → Sync → Import from OneNote (OneDrive)** → sign in with the code shown → pick notebooks.
Pages, subpages, pictures, to-dos and tags (Important / Question / Idea) come across. Importing again later adds new pages and updates
changed ones, but never overwrites pages you've edited in Feynos Notes or deleted. Ink drawings and file attachments aren't imported.

## docker-compose.yml

```yaml
services:
  feynos-notes-sync:
    image: ghcr.io/feynos-lin/feynos-note-sync:latest
    container_name: feynos-notes-sync
    restart: unless-stopped
    ports:
      - "8787:8787"
    environment:
      FEYNOS_PASSWORD: "change-me-to-something-long"
      TRUST_PROXY: "true"
      # BASE_PATH: "/notes"
      MAX_BODY_MB: "200"
      # MS_CLIENT_ID: ""
      # MS_TENANT: "common"
    volumes:
      - ./feynos-notes-data:/data
```

## Android signing (optional)

Add these repository secrets so every APK is signed with the same key and updates install over the old app:
`ANDROID_KEYSTORE_B64`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD`.
