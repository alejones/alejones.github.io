---
title: 'Converting docker-compose to quadlet'
description: ''
pubDate: 'May 30 2025'
heroImage: '/blog-placeholder-1.jpg'
---
# Converting Docker Compose to Podman Quadlets

Podman Quadlets provide systemd integration for containers without requiring a daemon. This guide converts a Docker Compose service to a user-specific Quadlet with GitHub Container Registry authentication.

## Starting Docker Compose File

```yaml
services:
  mailpiece-annotator:
    image: mailpiece-annotator:latest
    ports:
      - "8502:8501"
    volumes:
      - /home/ajones/avinas/userdata/users/ajones/annotator_data/Data:/app/Data
      - /home/ajones/laxnas:/app/laxnas
    environment:
      - PYTHONUNBUFFERED=1
    restart: unless-stopped
```

## Create the Quadlet File

Create the directory and file:

```bash
mkdir -p ~/.config/containers/systemd
```

Create `~/.config/containers/systemd/mailpiece-annotator.container`:

```ini
# ~/.config/containers/systemd/mailpiece-annotator.container

[Unit]
Description=Mailpiece Annotator Service
Wants=network-online.target
After=network-online.target
RequiresMountsFor=%t/containers

[Container]
Image=ghcr.io/alejones/llava-annotation-tool:latest
Pull=always
PublishPort=8502:8501
Volume=/home/ajones/avinas/userdata/users/ajones/annotator_data/Data:/app/Data
Volume=/home/ajones/laxnas:/app/laxnas
Environment=PYTHONUNBUFFERED=1
Restart=unless-stopped

[Service]
Restart=always
TimeoutStartSec=900

[Install]
WantedBy=multi-user.target default.target
```

## Key Differences

| Docker Compose | Quadlet |
|----------------|---------|
| `ports` | `PublishPort` |
| `volumes` (array) | `Volume` (one per line) |
| `environment` | `Environment` |
| `restart` | Handled by `[Service]` section |

## Enable User Lingering

Allow your user services to run without being logged in:

```bash
sudo loginctl enable-linger $USER
```

Verify lingering is enabled:

```bash
loginctl show-user $USER | grep Linger
```

## Authenticate with GitHub Container Registry

Create a GitHub Personal Access Token with `read:packages` scope at: GitHub → Settings → Developer settings → Personal access tokens

Login to GHCR:

```bash
podman login ghcr.io
```

Use your GitHub username and the PAT as the password.

## Deploy the Service

Reload systemd and start the service:

```bash
systemctl --user daemon-reload
systemctl --user enable --now mailpiece-annotator.service
```

Check the status:

```bash
systemctl --user status mailpiece-annotator.service
```

View logs:

```bash
journalctl --user -u mailpiece-annotator.service -f
```

## Manage the Service

Stop the service:

```bash
systemctl --user stop mailpiece-annotator.service
```

Restart the service:

```bash
systemctl --user restart mailpiece-annotator.service
```

Disable the service:

```bash
systemctl --user disable mailpiece-annotator.service
```

## Notes

- The `Pull=always` option pulls the latest image on each start
- User services with lingering start automatically at boot
- Authentication credentials persist in `~/.config/containers/auth.json`
- Quadlets automatically generate systemd service files from `.container` files