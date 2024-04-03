---
title: "Guacamole traefik docker compose"
date: 2023-11-05T00:00:00+00:00
tags:
  - docker
  - docker compose
  - traefik
  - guacamole
  - remote desktop
  - rds
  - vnc
---

## Guacamole

[Guacamole](https://guacamole.apache.org/) "Apache Guacamole is a clientless remote desktop gateway. It supports standard protocols like VNC, RDP, and SSH."

1. Set a A record for your domain pointing to the ip address where you are running this (e.g. guacamole.yourdomain.com).  
2. Add an .env file and a compose.yml
3. Run `$docker compose up -d`  

.env  
```bash
DOMAIN=my-domain.com
EMAIL=myemail@example-domain.com
```
compose.yml  
```yml
services:
  traefik:
    image: "traefik:v2.10.5"
    container_name: traefik
    command:
      #- --log.level=DEBUG
      #- --accesslog=true
      #- --accesslog.filePath=/logs/access.log
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --entrypoints.web.http.redirections.entrypoint.to=websecure
      - --entrypoints.web.http.redirections.entrypoint.scheme=https
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --api.dashboard=true
      - --certificatesresolvers.myresolver.acme.email=${EMAIL}
      - --certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json
      - --certificatesresolvers.myresolver.acme.tlschallenge=true
      #- --certificatesresolvers.myresolver.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory
      # uncomment to use staging server, production is default
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /traefik-letsencrypt:/letsencrypt
      - /traefik-logs:/logs
    #labels:
      #- "traefik.enable=true"
      #- "traefik.http.routers.dashboard.rule=Host(`traefik.${DOMAIN}`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))"
      #- "traefik.http.routers.dashboard.tls=true"
      #- "traefik.http.routers.dashboard.tls.certresolver=myresolver"
      #- "traefik.http.routers.dashboard.service=api@internal"
      #- "traefik.http.routers.dashboard.middlewares=auth"
      #- "traefik.http.middlewares.auth.basicauth.users=${BASIC_AUTH}"


  guacd:
    image: guacamole/guacd
    container_name: guacd
    hostname: guacd
    restart: unless-stopped
    volumes:
      - /root/guacd/drive:/drive
      - /root/guacd/record:/record


  guacamole-init:
    image: guacamole/guacamole:1.5.3
    user: root
    container_name: guacamole-init
    entrypoint: bash -c '/opt/guacamole/bin/initdb.sh --postgresql > /tmp/guac/init.sql'
    volumes:
      - /root/guac/init:/tmp/guac

  
  guacamole:
    image: guacamole/guacamole:1.5.3
    container_name: guacamole
    hostname: guacamole
    restart: unless-stopped
    depends_on:
      - guacd
      - guacamole-db
    environment:
      GUACD_HOSTNAME: guacd
      POSTGRESQL_HOSTNAME: guacamole-db
      POSTGRESQL_DATABASE: guacamole_db
      POSTGRESQL_USER: guacamole_user
      POSTGRESQL_PASSWORD: ${GUACAMOLE_PASSWORD}
    volumes:
      - /root/guac/init:/tmp/guac
    labels:
      - traefik.enable=true
      - traefik.http.routers.guacamole.rule=Host(`guac.${DOMAIN}`)
      - traefik.http.routers.guacamole.service=guacamole
      - traefik.http.routers.guacamole.entrypoints=websecure
      - traefik.http.routers.guacamole.tls=true
      - traefik.http.routers.guacamole.tls.certresolver=myresolver
      - traefik.http.services.guacamole.loadBalancer.server.port=8080
      - traefik.http.middlewares.guacprefix.addprefix.prefix=/guacamole
      - traefik.http.routers.guacamole.middlewares=guacprefix


  guacamole-db:
    image: postgres:13.4-buster
    container_name: guacamole-db
    hostname: guacamole-db
    depends_on: 
      - guacamole-init
    environment:
      POSTGRES_USER: guacamole_user
      POSTGRES_PASSWORD: '${GUACAMOLE_PASSWORD}'
      POSTGRES_DB: guacamole_db
      PGDATA: /var/lib/postgresql/data/guacamole
    restart: unless-stopped
    volumes:
      - /root/guac/init:/docker-entrypoint-initdb.d
      - /root/guac/-pgdata:/var/lib/postgresql/data
      - /root/guac/-data/database:/home/postgres/pgdata/data
```
