---
title: 'Move from docker-compose to quadlet'
description: ''
pubDate: 'May 30 2025'
heroImage: '/4seals.webp'
---
# Why do this?
For the applications I run, a full Kubernetes setup would be overkill, but I want something more robust than Podman Compose. I think Quadlets is a sweet spot for ease of use and reliability.

Quadlets integrate with systemd to automatically start and restart containers, giving me the reliability I need without the complexity. So far this setup has been very reliable for me.

My typical workflow starts with a Docker Compose file for quick testing and development. Once I'm ready for something long running, I convert it to a quadlet.

# What am I doing
This guide walks through converting a Docker Compose service to a Podman Quadlet with systemd integration. I'll cover authentication with GitHub Container Registry and set up user-specific services that start automatically on boot.

## Starting Docker Compose File
This is a dummy compose file. It's very similar to what I usually use for simple Python apps.

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

```ini
[Container]
Image=ghcr.io/alejones/my_container:latest
AutoUpdate=registry
PublishPort=8501:8501
Volume=%h/Data:/app/Data
Environment=PYTHONUNBUFFERED=1

[Service]
Restart=always

[Install]
WantedBy=default.target
```
#### Notes 
- Use %h instead of absolute paths like /home/username/ for portability
- AutoUpdate=registry enables automatic image updates with podman auto-update


## Key Differences

| Docker Compose | Quadlet |
|----------------|---------|
| `ports` | `PublishPort` |
| `volumes` | `Volume` (one per line) |
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
ajones@vm1:~$ loginctl show-user $USER | grep Linger
Linger=yes
```

## Authenticate with GitHub Container Registry

Create a GitHub Personal Access Token with `read:packages` scope at: GitHub → Settings → Developer settings → Personal access tokens

Login to GHCR:

```bash
podman login ghcr.io
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

You should see
```bash
ajones@vm1:~$ systemctl --user is-enabled my_container.service
generated
```

## Updating Images

Check for and apply image updates:

```bash
podman auto-update
```

This pulls newer images and restarts containers that have `AutoUpdate=registry` set.

If there is an update, you'll see the container getting pulled and a note that it has been updated.

```bash
podman auto-update
Trying to pull ghcr.io/alejones/myRepo/my_container:latest...
Getting image source signatures
Copying blob 8045c9806d81 skipped: already exists  
Copying blob 49ccfcf26a76 skipped: already exists  
Copying blob 61320b01ae5e skipped: already exists  
Copying blob 7a1cb8b88221 skipped: already exists  
Copying blob 6a3674a456ea skipped: already exists  
Copying blob 8991c9200d62 skipped: already exists  
Copying blob 157fcaa91dcb skipped: already exists  
Copying blob 88feadc186aa skipped: already exists  
Copying blob f15096e3c9ed skipped: already exists  
Copying blob 2505926d570d done   | 
Copying blob be1274d3cce0 skipped: already exists  
Copying config 47e63d279d done   | 
Writing manifest to image destination
            UNIT                         CONTAINER                                   IMAGE                                                   POLICY      UPDATED
            mailpiece-annotator.service  cb3948b23fb9 (systemd-mailpiece-annotator)  ghcr.io/alejones/someContainer:latest           registry    false
            slm-testing.service          7f427f80ba98 (systemd-slm-testing)          ghcr.io/alejones/myRepo/my_container:latest  registry    true
```
## Did it work?

You don't need to do any of these, but they might be helpful if you are running into trouble.

#### Check that the container is running

```bash
podman ps
```

#### Check the status:

```bash
systemctl --user status my_container.service
```

You should get response like this. I'm running a streamlit app, the output will change depending on your container.
```bash
ajones@vm1:~$ systemctl --user status my_container.service
● my_container.service
     Loaded: loaded (/home/ajones/.config/containers/systemd/my_container.container; generated)
     Active: active (running) since Fri 2025-05-30 17:27:39 CDT; 19min ago
   Main PID: 19876 (conmon)
      Tasks: 38 (limit: 76532)
     Memory: 217.0M (peak: 221.0M)
        CPU: 4.737s
     CGroup: /user.slice/user-1000.slice/user@1000.service/app.slice/my_container.service
             ├─libpod-payload-cb3948b23fb910ac512f13510c101a798129a77225e3b2fe4be19afdbbb9b055
             │ └─19888 /usr/local/bin/python3 /usr/local/bin/streamlit run app.py --server.address=0.0.0.0 --server.po>
             └─runtime
               ├─19854 /usr/bin/slirp4netns --disable-host-loopback --mtu=65520 --enable-sandbox --enable-seccomp --en>
               ├─19856 rootlessport
               ├─19862 rootlessport-child
               └─19876 /usr/bin/conmon --api-version 1 -c cb3948b23fb910ac512f13510c101a798129a77225e3b2fe4be19afdbbb9>

May 30 17:27:39 vm1 podman[19834]: 2025-05-30 17:27:39.633683542 -0500 CDT m=+0.020642742 image pull 1a42d82c72d>
May 30 17:27:39 vm1 podman[19834]: 2025-05-30 17:27:39.812141436 -0500 CDT m=+0.199100612 container init cb3948b>
May 30 17:27:39 vm1 podman[19834]: 2025-05-30 17:27:39.818534127 -0500 CDT m=+0.205493308 container start cb3948>
May 30 17:27:39 vm1 systemd[3138]: Started my_container.service.
May 30 17:27:39 vm1 my_container[19834]: cb3948b23fb910ac512f13510c101a798129a77225e3b2fe4be19afdbbb9b055
May 30 17:27:40 vm1 systemd-my_container[19876]: 
May 30 17:27:40 vm1 systemd-my_container[19876]:   You can now view your Streamlit app in your browser.
May 30 17:27:40 vm1 systemd-my_container[19876]: 
May 30 17:27:40 vm1 systemd-my_container[19876]:   URL: http://0.0.0.0:8501
May 30 17:27:40 vm1 systemd-my_container[19876]: 

```


#### View logs:

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

# Next thing I want to try
Podlet is a tool to automatically turn compose files into quadlets. These are my quick notes to try later. Follow at your own peril.


Podlet handles many of the tedious conversion details automatically and can generate multiple related files at once. It's especially useful for complex setups with multiple containers, networks, and volumes.

## Convert Compose to Quadlet

```bash
# Convert your compose file directly
podlet compose docker-compose.yml --file ~/.config/containers/systemd/

# Or pipe a command to create a quadlet
podlet podman run --name my-app -p 8501:8501 my-image:latest
```

