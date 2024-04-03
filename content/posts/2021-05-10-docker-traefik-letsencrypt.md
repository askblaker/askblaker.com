---
title: "HTTPS with letsencrypt using traefik and docker"
date: 2021-05-10T17:26:57+02:00
showToc: true
tags:
  - docker
  - traefik
  - letsencrypt
---

# Prerequisites
1. A domain name and dns service
2. A host with docker and docker-compose with a public ip (regular VPS)
3. Some knowledge of docker and docker-compose

# Setting up
1. DNS propagation might take some time, so set your domain first.
In your domain settings, set an 
```
|   DOMAIN                     | RECORD TYPE |   TARGET    |
| ---------------------------  | ----------- | ----------- |
| subdomain.example.com        | A RECORD    | <SERVER-IP> |
| *.subdomain.example.com      | A RECORD    | <SERVER-IP> |
| nginx.subdomain.example.com  | A RECORD    | <SERVER-IP> |
```


Check that its working with e.g. "ping <your-ip>" or use an online tool. You should see the ip of your server when trying to access subdomain.example.com.

2. On the server create a docker-compose file. 
```bash
nano docker-compose.yml
```
# Traefik only
```yaml
version: '3'
services:
  traefik:
    image: "traefik:v2.3.4"
    container_name: traefik
    command:
      - --log.level=DEBUG
      - --accesslog=true
      - --entrypoints.web.address=:80
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
    ports:
      - "80:80"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

Now you can try to use a browser to access the ip adress of the server and see the result in the logs streaming from the docker container. Its also a good time to test your subdomain.example.com

```bash
# browser address field: <server-ip>
traefik    | 12.34.567.89 - - [10/May/2021:15:56:36 +0000] "GET / HTTP/1.1" - - "-" "-" 1 "-" "-" 0ms
# browser address field: <server-ip>/something
traefik    | 12.34.567.89 - - [10/May/2021:15:56:38 +0000] "GET /something HTTP/1.1" - - "-" "-" 1 "-" "-" 0ms
# browser address field: subdomain.example.com/something
traefik    | 12.34.567.89 - - [10/May/2021:15:56:42 +0000] "GET /something HTTP/1.1" - - "-" "-" 1 "-" "-" 0ms
# It works!
```

# Traefik + insecure dashboard
If you want, you can also check out the dashboard (warning, this is insecure mode)
```yaml
version: '3'
services:
  traefik:
    image: "traefik:v2.3.4"
    container_name: traefik
    command:
      - --log.level=DEBUG
      - --accesslog=true
      - --entrypoints.web.address=:80
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --api.dashboard=true
      - --api.insecure=true
    ports:
      - "8080:8080"
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

# Browser address bar:<ip-address>:8080/dashboard/
# Make sure to remember the trailing slash
```

# Traefik with secured dashboard
Or in secure mode with basic auth like this. Note this is regular http port, and username/password "test"
> From traefik docs:  
> Note: when used in docker-compose.yml all dollar signs in the hash need to be doubled for escaping.  
> To create user:password pair, it's possible to use this command:
```bash
echo $(htpasswd -nB user) | sed -e s/\\$/\\$\\$/g
```
```yaml
version: '3'
services:
  traefik:
    image: "traefik:v2.3.4"
    container_name: traefik
    command:
      - --log.level=DEBUG
      - --accesslog=true
      - --entrypoints.web.address=:80
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --api.dashboard=true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`subdomain.example.com`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=test:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/"

# Browser address bar:subdomain.example.com/dashboard/
# Make sure to remember the trailing slash
```
# Traefik with secured dashboard on sub sub domain
On a separate domain
```yaml
version: '3'
services:
  traefik:
    image: "traefik:v2.3.4"
    container_name: traefik
    command:
      - --log.level=DEBUG
      - --accesslog=true
      - --entrypoints.web.address=:80
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --api.dashboard=true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.subdomain.example.com`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=test:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/"

# Browser address bar: traefik.subdomain.example.com/dashboard/
# Make sure to remember the trailing slash
```

# With https certificate from letsencrypt
```yaml
version: '3'
services:
  traefik:
    image: "traefik:v2.3.4"
    container_name: traefik
    command:
      - --log.level=DEBUG
      - --accesslog=true
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --api.dashboard=true
      - --certificatesresolvers.myresolver.acme.email=email@example.com
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
      - "traefik.http.routers.dashboard.rule=Host(`traefik.subdomain.example.com`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))"
      - "traefik.http.routers.dashboard.tls=true"
      - "traefik.http.routers.dashboard.tls.certresolver=myresolver"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=test:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/"

volumes:
  cert-vol:

# Browser address bar: https://traefik.subdomain.example.com/dashboard/
# Make sure to remember the trailing slash
```

You should now be able to see in the logs that a certificate has been generated, and you are secured with https. If you check the certificate, you will se that it is issued to traefik.subdomain.example.com. But how would we add another service? Lets try adding a nginx test service, and see how easy it is.  

# Separate container with https
```yaml
version: '3'
services:
  traefik:
    image: "traefik:v2.3.4"
    container_name: traefik
    command:
      - --log.level=DEBUG
      - --accesslog=true
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --api.dashboard=true
      - --certificatesresolvers.myresolver.acme.email=email@example.com
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
      - "traefik.http.routers.dashboard.rule=Host(`traefik.subdomain.example.com`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))"
      - "traefik.http.routers.dashboard.tls=true"
      - "traefik.http.routers.dashboard.tls.certresolver=myresolver"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=test:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/"

  nginx:
    image: "nginx:latest"
    environment:
      - NGINX_PORT=6543
    labels:
      - traefik.enable=true
      - traefik.http.routers.nginx.rule=Host(`nginx.subdomain.example.com`)
      - traefik.http.routers.nginx.service=nginx-service
      - traefik.http.routers.nginx.entrypoints=websecure
      - traefik.http.routers.nginx.tls.certresolver=myresolver
      - traefik.http.routers.nginx.tls=true
      - traefik.http.services.nginx-service.loadbalancer.server.port=6543

volumes:
  cert-vol:

# Browser address bar: https://nginx.subdomain.example.com/
```


# Separate container using *.subdomain wildcard dns and non-80 port
```yaml
version: '3'
services:
  traefik:
    image: "traefik:v2.3.4"
    container_name: traefik
    command:
      - --log.level=DEBUG
      - --accesslog=true
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --api.dashboard=true
      - --certificatesresolvers.myresolver.acme.email=email@example.com
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
      - "traefik.http.routers.dashboard.rule=Host(`traefik.subdomain.example.com`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))"
      - "traefik.http.routers.dashboard.tls=true"
      - "traefik.http.routers.dashboard.tls.certresolver=myresolver"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=test:$$apr1$$H6uskkkW$$IgXLP6ewTrSuBkTrqE8wj/"

  echo:
    image: "hashicorp/http-echo"
    command:
      - --text="Hello There!"
      - --listen=:6543
    labels:
      - traefik.enable=true
      - traefik.http.routers.echo.rule=Host(`myname.subdomain.example.com`)
      - traefik.http.routers.echo.service=echo-service
      - traefik.http.routers.echo.entrypoints=websecure
      - traefik.http.routers.echo.tls.certresolver=myresolver
      - traefik.http.routers.echo.tls=true
      - traefik.http.services.echo-service.loadbalancer.server.port=6543
    
volumes:
  cert-vol:

# Browser address bar: https://echo.subdomain.example.com/
```

# Finally
As you play with this you probably notice the obscene amount of logs this is generating. When you no more need to log so much, remove or comment out loglevel and accesslog
```yaml
# - --log.level=DEBUG
# - --accesslog=true
```


