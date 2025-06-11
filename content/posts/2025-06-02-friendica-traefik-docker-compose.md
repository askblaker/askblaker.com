---
title: "Friendica traefik docker compose"
date: 2025-06-02T00:00:00+00:00
tags:
  - docker
  - docker compose
  - traefik
  - friendica
---

## Friendica

[Friendica](https://friend.apache.org/) "A Decentralized Social Network".

I wanted to try friendica, but it seems quite difficult to try without having every setting complitely corect. This compose will make it possible to just try ut with localhost, and use mailhog so you dont need a correct working email as well.

1. Set a A record for your domain pointing to the ip address where you are running this (e.g. friendica.yourdomain.com).  
2. Add an .env file and a compose.yml
3. Run `$docker compose up -d`  

compose.yml  
```yml
services:
  db:
    image: mariadb
    restart: always
    environment:
      - MYSQL_USER=friendica
      - MYSQL_PASSWORD=test
      - MYSQL_DATABASE=friendica
      - MYSQL_ROOT_PASSWORD=rootpw

  app:
    image: friendica
    restart: always
    ports:
      - 80:80
    environment:
      - MYSQL_HOST=db
      - MYSQL_USER=friendica
      - MYSQL_PASSWORD=test
      - MYSQL_DATABASE=friendica
      - FRIENDICA_ADMIN_MAIL=root@friendica.local
      - FRIENDICA_DEBUGGING=true
      - FRIENDICA_LOGLEVEL=debug
      - FRIENDICA_URL=http://localhost
      - FRIENDICA_NO_VALIDATION=true
      - SMTP=mailhog
      - SMTP_FROM=demo
      - SMTP_PORT=1025
      - SMTP_DOMAIN=friendica.local
      - SMTP_TLS=off
      - SMTP_STARTTLS=off
      - SMTP_AUTH=off
    depends_on:
      - db

  mailhog:
    image: mailhog/mailhog:v1.0.1
    restart: always
    ports:
      - 8025:8025
```
