---
title: "Self hosted taguette"
date: 2023-05-11T22:08:00+00:00
tags:
  - docker
  - docker-compose
  - vps
  - taguette
---

## Taguette

Free and open-source qualitative research tool (which works on all operating systems!)

Check it out [here](https://www.taguette.org)

1. Set an A record for every subdomain (or use wildcard) from your domain pointing to the server iå.
2. Install docker + docker compose
3. Copy files below
4. Edit as needed
5. Run `docker compose up`

## docker-compose.yaml

```yaml
version: "3"
services:
  traefik:
    image: "traefik:v2.3.4"
    container_name: traefik
    command:
      #  --log.level=DEBUG
      #  --accesslog=true
      # uncomment for logs
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --entrypoints.web.http.redirections.entrypoint.to=websecure
      - --entrypoints.web.http.redirections.entrypoint.scheme=https
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --api.dashboard=true
      - --certificatesresolvers.myresolver.acme.email=${LE_EMAIL}
      - --certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json
      - --certificatesresolvers.myresolver.acme.tlschallenge=true

    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - cert-vol:/letsencrypt
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.${DOMAIN}`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))"
      - "traefik.http.routers.dashboard.tls=true"
      - "traefik.http.routers.dashboard.tls.certresolver=myresolver"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=auth"
      #- "traefik.http.middlewares.auth.basicauth.users=testuser:$$2y$$05$$7CsWRvfQVTS0y.8UI9Pa4u2RvwSKmkcYSnAmiXxCRgPYo3elZapIK"
      # replace the testuser:password using: echo $(htpasswd -nB testuser) | sed -e s/\\$/\\$\\$/g
      # then uncomment to add basic auth

  taguette:
    image: "quay.io/remram44/taguette:1.4.1"
    command: ["server", "/data/config.py"]
    labels:
      - traefik.enable=true
      - traefik.http.routers.echo.rule=Host(`taguette.${DOMAIN}`)
      - traefik.http.routers.echo.service=taguette-service
      - traefik.http.routers.echo.entrypoints=websecure
      - traefik.http.routers.echo.tls.certresolver=myresolver
      - traefik.http.routers.echo.tls=true
      - traefik.http.services.taguette-service.loadbalancer.server.port=7465
    volumes:
      - "./data:/data"

  mailhog:
    image: mailhog/mailhog:v1.0.1
    labels:
      - traefik.enable=true
      - traefik.http.routers.mailhog.rule=Host(`mailslurper.${DOMAIN}`)
      - traefik.http.routers.mailhog.service=mailhog-service
      - traefik.http.routers.mailhog.entrypoints=websecure
      - traefik.http.routers.mailhog.tls.certresolver=myresolver
      - traefik.http.routers.mailhog.tls=true
      - traefik.http.services.mailhog-service.loadbalancer.server.port=8025
      - traefik.http.routers.mailhog.middlewares=auth
      #- "traefik.http.middlewares.auth.basicauth.users=testuser:$$2y$$05$$7CsWRvfQVTS0y.8UI9Pa4u2RvwSKmkcYSnAmiXxCRgPYo3elZapIK"
      # replace the testuser:password using: echo $(htpasswd -nB testuser) | sed -e s/\\$/\\$\\$/g
      # then uncomment to add basic auth
```

## .env

```bash
LE_EMAIL=your@email.here
DOMAIN=yourdomain.com
```
