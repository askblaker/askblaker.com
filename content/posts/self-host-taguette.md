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

1. Set an A record for all three subdomains (taguette, mailhog, traefik) pointing to the server ip. Wait a bit for it to propagate.
2. Install docker + docker compose
3. Go to your desired folder and create docker-compose.yaml, .env, and ./data/config.py with the content below
4. Set email, domain, and basic auth in .env  
   Note: For user:password pair, you can use this command:  
   `echo $(htpasswd -nB testuser) | sed -e s/\\$/\\$\\$/g`
5. Set cookie secret in config.py (and any other desired settings)  
   To generate a cookie secreet you can use this command:  
   `openssl rand -base64 48`
6. Run `docker compose up traefik`
7. Ensure that traefik.${DOMAIN} works
8. In a different terminal, run `docker compose run --rm -it taguette` to set password
9. Ensure taguette.${DOMAIN} works and you can log in with admin user
10. Shut down both containers
11. Comment out the letsencrypt staging urls and debug and auth logging for traefik
12. Run `docker compose up -d`
13. Enjoy / debug

## docker-compose.yaml

```yaml
version: "3"
services:
  traefik:
    image: "traefik:v2.3.4"
    container_name: traefik
    command:
      - --log.level=DEBUG
      - --accesslog=true
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
      - "--certificatesresolvers.myresolver.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`traefik.${DOMAIN}`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))"
      - "traefik.http.routers.dashboard.tls=true"
      - "traefik.http.routers.dashboard.tls.certresolver=myresolver"
      - "traefik.http.routers.dashboard.service=api@internal"
      - "traefik.http.routers.dashboard.middlewares=auth"
      - "traefik.http.middlewares.auth.basicauth.users=${BASIC_AUTH}"

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
      - traefik.http.routers.mailhog.rule=Host(`mailhog.${DOMAIN}`)
      - traefik.http.routers.mailhog.service=mailhog-service
      - traefik.http.routers.mailhog.entrypoints=websecure
      - traefik.http.routers.mailhog.tls.certresolver=myresolver
      - traefik.http.routers.mailhog.tls=true
      - traefik.http.services.mailhog-service.loadbalancer.server.port=8025
      - traefik.http.routers.mailhog.middlewares=auth
      - "traefik.http.middlewares.auth.basicauth.users=${BASIC_AUTH}"
```

## config.py

```python
# This is the configuration file for Taguette
# It is a Python file, so you can use the full Python syntax

# Name of this server
NAME = "Misconfigured Taguette Server"

# Address and port to listen on
BIND_ADDRESS = "0.0.0.0"
PORT = 7465

# Base path of the application
BASE_PATH = "/"

# A unique secret key that will be used to sign cookies
# set this to something secure and uncomment
# you can e.g. use the command 'openssl rand -base64 48'
SECRET_KEY = "<insert secret key here>"

# Database to use
# This is a SQLAlchemy connection URL; refer to their documentation for info
# https://docs.sqlalchemy.org/en/latest/core/engines.html
# If using SQLite3 on Unix, note the 4 slashes for an absolute path
# (keep 3 before a relative path)
DATABASE = "sqlite:////data/database.sqlite3"

# Redis instance for live collaboration
# This is not required if using a single server, collaboration will still work
#REDIS_SERVER = 'redis://localhost:6379'

# Address to send system emails from
EMAIL = "Misconfigured Taguette Server <taguette@example.com>"

# Terms of service (HTML file)
TOS_FILE = None #'tos.html'
# If set to None, no terms of service link will be displayed anywhere
#TOS_FILE = None

# Extra footer at the bottom of every page
#EXTRA_FOOTER = """
#  | This instance of Taguette is managed by Example University.
#  Please <a href="mailto:it@example.org">email IT</a> with any questions.
#"""

# Default language
DEFAULT_LANGUAGE = 'en_US'

# SMTP server to use to send emails
MAIL_SERVER = {
    "ssl": False,
    "host": "mailhog",
    "port": 1025,
}

# Whether users must explicitly accept cookies before using the website
COOKIES_PROMPT = False

# Whether new users can create an account
REGISTRATION_ENABLED = True

# Whether users can import projects from SQLite3 files
SQLITE3_IMPORT_ENABLED = True

# Set this to true if you are behind a reverse proxy that sets the
# X-Forwarded-For header.
# Leave this at False if users are connecting to Taguette directly
X_HEADERS = True

# Time limits for converting documents
CONVERT_TO_HTML_TIMEOUT = 3 * 60  # 3min for importing document into Taguette
CONVERT_FROM_HTML_TIMEOUT = 3 * 60  # 3min for exporting from Taguette

# If you want to export metrics using Prometheus, set a port number here
#PROMETHEUS_LISTEN = "0.0.0.0:9101"

# If you want to report errors to Sentry, set your DSN here
#SENTRY_DSN = "https://<key>@sentry.io/<project>"
```

## .env

```bash
LE_EMAIL=your@email.here
DOMAIN=yourdomain.com
BASIC_AUTH=testuser:$$2y$$05$$ZWEu/Xe2Xw9uAXpHKKefnudGbzKLLr/c7mnc1cL/UYXTr/k/dotgq
```
