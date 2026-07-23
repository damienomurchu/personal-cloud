# Open WebUI — Local Mac mini Service

This directory contains the Docker Compose service definition for running **Open WebUI** locally on the Mac mini.

Open WebUI runs in a container.  
Ollama runs directly on the Mac host.  
External access is handled separately through **Cloudflare Tunnel + Cloudflare Access**.

The public URL is intended to be:

```text
https://ai.damienmurphy.net
```

---

## Architecture

```text
Browser
  ↓
Cloudflare Access
  ↓
Cloudflare Tunnel
  ↓
Mac mini localhost:3000
  ↓
Open WebUI container
  ↓
host.docker.internal:11434
  ↓
Ollama running on Mac mini
```

The service is intentionally bound to localhost only:

```yaml
ports:
  - "127.0.0.1:${OPEN_WEBUI_PORT}:8080"
```

This means Open WebUI is **not directly exposed to the LAN or internet**. Cloudflare Tunnel is expected to proxy to it locally.

---

## Files

```text
docker-compose.yml   # Defines the Open WebUI container
.env                 # Local configuration values used by Compose
README.md            # This file
```

Expected wider layout:

```text
~/code/
  apps/
    open-webui/
      docker-compose.yml
      .env
      README.md

  data/
    open-webui/
      # Persistent Open WebUI data

  backups/
    open-webui/
      # Backup archives
```

---

## `.env`

Example:

```env
OPEN_WEBUI_PORT=3000
OPEN_WEBUI_DATA_DIR=/Users/damien/code/data/open-webui

WEBUI_AUTH=true
ENABLE_SIGNUP=false
WEBUI_URL=https://ai.damienmurphy.net

OLLAMA_BASE_URL=http://host.docker.internal:11434
```

### Important values

#### `OPEN_WEBUI_PORT`

The local host port Open WebUI is exposed on.

```env
OPEN_WEBUI_PORT=3000
```

Local access:

```text
http://localhost:3000
```

Cloudflare Tunnel should point to:

```text
http://localhost:3000
```

---

#### `OPEN_WEBUI_DATA_DIR`

Where Open WebUI stores its persistent data.

```env
OPEN_WEBUI_DATA_DIR=/Users/damien/code/data/open-webui
```

This is important. It contains things like users, conversations, app state, uploaded files, and configuration.

Do **not** casually delete this directory unless you are deliberately resetting Open WebUI.

---

#### `WEBUI_AUTH`

```env
WEBUI_AUTH=true
```

Keeps Open WebUI login enabled.

Even though Cloudflare Access sits in front of the app, Open WebUI still has its own login. This gives two layers:

```text
Cloudflare Access login
→ Open WebUI login
```

This is intentional.

---

#### `ENABLE_SIGNUP`

```env
ENABLE_SIGNUP=false
```

Prevents arbitrary users from signing themselves up.

New users should be created/managed from the Open WebUI admin panel.

---

#### `WEBUI_URL`

```env
WEBUI_URL=https://ai.damienmurphy.net
```

The external URL for the service.

Used by Open WebUI when it needs to know its public-facing address.

---

#### `OLLAMA_BASE_URL`

```env
OLLAMA_BASE_URL=http://host.docker.internal:11434
```

Open WebUI runs in a container, but Ollama runs directly on the Mac mini host.

From inside the container, `localhost` would refer to the container itself, not the Mac.

So this uses:

```text
host.docker.internal
```

to reach Ollama on the Mac host.

---

## `docker-compose.yml`

Expected service shape:

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped

    ports:
      - "127.0.0.1:${OPEN_WEBUI_PORT}:8080"

    environment:
      WEBUI_AUTH: "${WEBUI_AUTH}"
      ENABLE_SIGNUP: "${ENABLE_SIGNUP}"
      WEBUI_URL: "${WEBUI_URL}"
      OLLAMA_BASE_URL: "${OLLAMA_BASE_URL}"

    volumes:
      - "${OPEN_WEBUI_DATA_DIR}:/app/backend/data"
```

### Key points

The container listens internally on:

```text
8080
```

The Mac exposes it locally on:

```text
127.0.0.1:3000
```

The persistent Open WebUI data lives on the host at:

```text
~/code/data/open-webui
```

and is mounted into the container at:

```text
/app/backend/data
```

The container can be destroyed/recreated safely as long as the data directory is preserved.

Containers are disposable. The data directory is not.

---

## Common commands

Run from this directory:

```bash
cd ~/code/apps/open-webui
```

### Start

```bash
docker compose up -d
```

### Stop

```bash
docker compose down
```

### Restart

```bash
docker compose restart
```

### View logs

```bash
docker logs -f open-webui
```

### Check status

```bash
docker ps --filter "name=open-webui"
```

### Update Open WebUI

```bash
docker compose pull
docker compose up -d
docker image prune -f
```

Before updating, make a backup.

---

## Ollama checks

Ollama should be running on the Mac host.

Check from the Mac:

```bash
curl http://localhost:11434/api/tags
```

Check from inside the container:

```bash
docker exec -it open-webui curl http://host.docker.internal:11434/api/tags
```

If the second command fails, Open WebUI cannot reach Ollama.

---

## Cloudflare Tunnel

Cloudflare Tunnel should route:

```text
ai.damienmurphy.net
→ http://localhost:3000
```

This service should **not** be exposed directly with public ports or router port forwarding.

Correct:

```text
Cloudflare Tunnel → localhost:3000
```

Wrong:

```text
Public internet → Mac mini port 3000
```

---

## Cloudflare Access

Cloudflare Access should protect:

```text
https://ai.damienmurphy.net
```

Expected login flow:

```text
Cloudflare Access
→ Open WebUI login
→ Chat UI
```

Access policy should normally allow only explicit trusted email addresses.

Do not use:

```text
Bypass everyone
```

unless deliberately testing.

---

## User and model access

Open WebUI users need two things:

1. Their role must be `User`, not `Pending`.
2. The models must be visible/shared to them.

The default permission toggle:

```text
Models Access
```

only allows users to use the models feature. It does **not** automatically grant access to every backend Ollama model.

To share models:

```text
Admin / Workspace
→ Models
→ Select model
→ Set visibility/access
```

For simple trusted usage, make selected small models public/shared.

Suggested normal-user models:

```text
qwen2.5-coder:7b
mistral-small
```

Keep large/experimental models private unless deliberately sharing them.

---

## Backups

The important directory is:

```text
~/code/data/open-webui
```

Back it up before upgrades or major configuration changes.

Example backup:

```bash
mkdir -p ~/code/backups/open-webui

tar -czf ~/code/backups/open-webui/open-webui-$(date +%Y%m%d-%H%M%S).tar.gz \
  -C ~/code/data/open-webui .
```

Restore pattern:

```bash
cd ~/code/apps/open-webui
docker compose down

mv ~/code/data/open-webui ~/code/data/open-webui.broken.$(date +%Y%m%d-%H%M%S)
mkdir -p ~/code/data/open-webui

tar -xzf ~/code/backups/open-webui/<backup-file>.tar.gz \
  -C ~/code/data/open-webui

docker compose up -d
```

---

## Troubleshooting

### Open WebUI does not load locally

Check:

```bash
docker ps --filter "name=open-webui"
docker logs -f open-webui
curl -I http://localhost:3000
```

---

### Open WebUI loads locally but not through Cloudflare

Check the tunnel route:

```text
ai.damienmurphy.net
→ http://localhost:3000
```

Check that the tunnel connector is healthy in Cloudflare.

---

### Open WebUI loads but shows no models

Check Ollama:

```bash
curl http://localhost:11434/api/tags
```

Check from container:

```bash
docker exec -it open-webui curl http://host.docker.internal:11434/api/tags
```

Then check Open WebUI admin/settings connections.

---

### Other users see no models

Check:

```text
Admin Panel → Users
```

User role should be:

```text
User
```

not:

```text
Pending
```

Then check:

```text
Workspace → Models
```

and ensure the relevant models are public/shared with that user or group.

---

## Things not to casually change

Do not change this unless you know why:

```yaml
127.0.0.1:${OPEN_WEBUI_PORT}:8080
```

Binding to `127.0.0.1` is intentional.

Do not remove this volume:

```yaml
${OPEN_WEBUI_DATA_DIR}:/app/backend/data
```

That is what keeps the app data alive across container recreations.

Do not expose the service directly to the internet.

Do not rely on anonymous Docker volumes for anything important.

---

## Mental model

This is a local-first AI service.

Cloudflare provides the safe front door.

Open WebUI provides the app.

Ollama provides the models.

Docker runs the UI.

The data directory is the crown jewels.

Everything else is plumbing.
