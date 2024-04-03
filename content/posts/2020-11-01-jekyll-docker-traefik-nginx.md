---
title: "Jekyll with Docker, Nginx and Traefik"
date: 2020-11-01T12:00:00+00:00
tags:
  - jekyll
  - nginx
  - traefik
  - letsencrypt
  - docker
  - docker-compose
---

## Prerequisites

1. A [hardened]({{< ref "/posts/hardening-ubuntu.md" >}} "hardened") VPS
2. A domain with dns service. (Forward example.com to your vps ip)
3. [Jekyll](https://jekyllrb.com/)

```bash
   > sudo apt-get install ruby-full build-essential zlib1g-dev

   > echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc  
   > echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc  
   > echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc  
   > source ~/.bashrc

   > sudo gem install jekyll bundler

   > cd jekyll-directory  
   > sudo bundle install  
   > jekyll build
```
4. A nice theme. This is, and I recommend: [minimal-mistakes](https://mmistakes.github.io/minimal-mistakes/)
5. A folder structure like this:

```bash
.
├── jekyll-folder
├── docker-compose.yml
├── .env
```

6. A docker-compose.yml about like this:

```
version: "3.8"

services:
  web:
    image: nginx
    container_name: nginx
    volumes:
      - ./templates:/etc/nginx/templates
      - ./www/_site:/usr/share/nginx/html/askblaker
      - ./www_nginx.conf:/etc/nginx/conf.d/www_nginx.conf
    environment:
      - NGINX_HOST=${HOSTNAME}
      - NGINX_PORT=80
    labels:
      - traefik.enable=true
      - traefik.http.routers.nginx.rule=Host(`${HOSTNAME}`)
      - traefik.http.routers.nginx.service=nginx-service
      - traefik.http.routers.nginx.entrypoints=websecure
      - traefik.http.routers.nginx.tls.certresolver=basic
      - traefik.http.routers.nginx.tls=true
      - traefik.http.routers.nginx.tls.domains[0].main=${HOSTNAME}
      - traefik.http.routers.nginx.tls.domains[0].sans=*.${HOSTNAME}
      - traefik.http.services.nginx-service.loadbalancer.server.port=80

  traefik:
    image: "traefik:v2.2"
    container_name: traefik
    command:
      - --log.level=DEBUG
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --certificatesresolvers.basic.acme.email=${LE_EMAIL}
      - --certificatesresolvers.basic.acme.storage=/letsencrypt/acme.json
      - --certificatesresolvers.basic.acme.dnschallenge.provider=digitalocean
      - --api
    environment:
      - DO_AUTH_TOKEN=${DO_AUTH_TOKEN}
    ports:
      - "80:80"
      - "443:443"
    labels:
      traefik.enable: true
      # Global redirection: http to https
      traefik.http.routers.http-catchall.rule: HostRegexp(`{host:(www\.)?.+}`)
      traefik.http.routers.http-catchall.entrypoints: web
      traefik.http.routers.http-catchall.middlewares: wwwtohttps

      # Global redirection: https (www.) to https
      traefik.http.routers.wwwsecure-catchall.rule: HostRegexp(`{host:(www\.).+}`)
      traefik.http.routers.wwwsecure-catchall.entrypoints: websecure
      traefik.http.routers.wwwsecure-catchall.tls: true
      traefik.http.routers.wwwsecure-catchall.middlewares: wwwtohttps

      # middleware: http(s)://(www.) to  https://
      traefik.http.middlewares.wwwtohttps.redirectregex.regex: ^https?://(?:www\.)?(.+)
      traefik.http.middlewares.wwwtohttps.redirectregex.replacement: https://$${1}
      traefik.http.middlewares.wwwtohttps.redirectregex.permanent: true
    volumes:
      - cert-vol:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro

volumes:
  cert-vol:
```

7. And a .env file:

```

HOSTNAME=example.com
LE_EMAIL=yourname@example.com
DO_AUTH_TOKEN=your_digitalocean_api_token_goes_here

```

8. If you dont have it yet, install [docker](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04) and [docker-compose](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-compose-on-ubuntu-20-04)

```bash
> sudo apt update
> sudo apt install apt-transport-https ca-certificates curl software-properties-common
> curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

> sudo add-apt-repository \"deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable\"

> sudo apt update
> apt-cache policy docker-ce
> sudo apt install docker-ce
> sudo usermod -aG docker ${USER}
> su - ${USER}

> sudo curl -L \"https://github.com/docker/compose/releases/download/1.27.4/docker-compose-$(uname -s)-$(uname -m)\" -o /usr/local/bin/docker-compose

> sudo chmod +x /usr/local/bin/docker-compose

```

