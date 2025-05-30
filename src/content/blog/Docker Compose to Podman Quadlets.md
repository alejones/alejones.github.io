---
title: 'Converting docker-compose to quadlet'
description: ''
pubDate: 'May 30 2025'
heroImage: '/4seals.webp'
---
# Why do this?
For the applications I run, a full k8s set up would be overkill. Yet, I'd like something more robust than using Podman Compose. For these cases I reach for quadlets.
This allows for my containers to automatically start up, and restart. So far this set up has been very reliable for me.

Despite this I usually start with writing a docker compose file for quick testing. These are the steps I take to move to quadlets.

# What am I doing

Podman Quadlets provide systemd integration for containers without requiring a daemon. This guide converts a Docker Compose service to a user-specific Quadlet with GitHub Container Registry authentication.

## Starting Docker Compose File

```yaml
services:
  my_container:
    image: my_container:latest
    ports:
      - "8501:8501"
    volumes:
      - /home/ajones/Data:/app/data
    environment:
      - PYTHONUNBUFFERED=1
    restart: unless-stopped
```

## Create the Quadlet File

Create the directory and file:

```bash
mkdir -p ~/.config/containers/systemd
```

Create and open the container file. 
```bash
 nano ~/.config/containers/systemd/my_container.container
```

Paste in the the following data
### Notes 
- Use %h instead of absolute paths like /home/username/ for portability
- AutoUpdate=registry enables automatic image updates with podman auto-update

```ini
# ~/.config/containers/systemd/my_container.container

[Container]
Image=ghcr.io/alejones/my_container:latest
AutoUpdate=registry
PublishPort=8501:8501
Volume=%h/avinas/Data:/app/Data
Environment=PYTHONUNBUFFERED=1

[Service]
Restart=always

[Install]
WantedBy=default.target
```

## Key Differences

| Docker Compose | Quadlet |
|----------------|---------|
| `ports` | `PublishPort` |
| `volumes` (array) | `Volume` (one per line) |
| `environment` | `Environment` |
| `restart` | Handled by `[Service]` section |
| Image updates | `AutoUpdate=registry` |


## Enable User Lingering

Allow your user services to run without being logged in:

```bash
sudo loginctl enable-linger $USER
```

Verify lingering is enabled:

```bash
loginctl show-user $USER | grep Linger
```

You should get an output like this
```bash
Linger=yes
```

## Authenticate with GitHub Container Registry

Create a GitHub Personal Access Token with `read:packages` scope at: GitHub → Settings → Developer settings → Personal access tokens

Login to GHCR:

```bash
ajones@vm1:~$ loginctl show-user $USER | grep Linger
Linger=yes
```

Use your GitHub username and the Access Token as the password.

## Deploy the Service

Reload systemd and start the service:

```bash
systemctl --user daemon-reload
```

```bash
systemctl --user start my_container.service
```

The service should auto-enable due to `WantedBy=default.target` in the quadlet. Verify it's enabled:

```bash
systemctl --user is-enabled my_container.service
```

## Did it work?

You don't need to do any of these, but they might be helpful if you are running into trouble.

Check the status:

```bash
systemctl --user status my_container.service
```

You should get this response
```bash
ajones@vm1:~$ systemctl --user is-enabled my_container.service
generated
```


View logs:

```bash
journalctl --user -u my_container.service -f
```

## Manage the Service

Stop the service:

```bash
systemctl --user stop my_container.service
```

Restart the service:

```bash
systemctl --user restart my_container.service
```

Disable the service:

```bash
systemctl --user disable my_container.service
```

## Notes

- The `Pull=always` option pulls the latest image on each start
- User services with lingering start automatically at boot
- Authentication credentials persist in `~/.config/containers/auth.json`
- Quadlets automatically generate systemd service files from `.container` files