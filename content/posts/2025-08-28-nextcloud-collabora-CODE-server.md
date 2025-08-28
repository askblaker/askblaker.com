---
title: "Nextcloud Collabora CODE server"
date: 2025-08-27T00:00:00+00:00
tags:
  - docker
  - fly.io
  - nextcloud
---

## Nextcloud

[Nextcloud](https://nextcloud.com/) "A selfhostable open source cloud / productivity suite".

I wanted to try using nextcloud. For the live editing document and spreadsheet functionality i needed to host a [Collabora CODE Server](https://sdk.collaboraonline.com/docs/installation/CODE_Docker_image.html)

This only needs to run when i use it, and preferably not incur any costs when not, so i decided to run it on fly.io, which has automatic shutdown. The required config was very short, and its runs very well.

```toml
# fly.toml
app = 'my-unique-CODE-server-name'
primary_region = 'fra'

[env]
# You need this to make the docker container bind to 0.0.0.0
LOOLWSD_LISTEN = "0.0.0.0"
# You need this if you want to control what spellcheck dictionaries to include
dictionaries = "en_US en_GB nb_NO"

[build]
[http_service]
  internal_port = 9980
  force_https = true
  auto_stop_machines = 'stop'
  auto_start_machines = true
  min_machines_running = 0
  processes = ['app']

[[vm]]
  memory = '1gb'
  cpu_kind = 'shared'
  cpus = 1
```

```Dockerfile
# Dockerfile
FROM collabora/code:25.04.4.2.1

# We will rely on fly.io to provide ssl termination
ENV DONT_GEN_SSL_CERT=1
ENV extra_params="--o:ssl.enable=false --o:ssl.termination=true"

EXPOSE 9980
```