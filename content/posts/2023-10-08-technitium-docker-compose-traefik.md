---
title: "Technitium docker compose traefik"
date: 2023-10-08T10:00:00+00:00
tags:
  - docker
  - docker compose
  - traefik
  - technitium
  - dns
---

## Technitium

[Technitium](https://technitium.com/dns/) is a self hostable open source dns server.

Self host a DNS server for privacy & security  

Block ads & malware at DNS level for your entire network!  

1. Set a A record for your domain pointing to the ip address where you are running this.  
2. Add an .env file and a compose.yml (docker compose picks up the .env file automatically)  
3. Run `$docker compose up -d`  

.env  
```bash
DOMAIN=my.example-domain.com
TRAEFIK_DOMAIN=traefik.example-domain.com
EMAIL=myemail@example-domain.com
```
compose.yml  
```yml
version: "3"
services:
  traefik:
    image: "traefik:v2.10.4"
    container_name: traefik
    command:
      #- --log.level=DEBUG
      #- --accesslog=true
      - --accesslog.filePath=/logs/access.log
      - --entrypoints.dnsudp.address=:53/udp
      - --entrypoints.dnstcp.address=:53
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --entrypoints.dnstls.address=:853
      - --entrypoints.web.http.redirections.entrypoint.to=websecure
      - --entrypoints.web.http.redirections.entrypoint.scheme=https
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --api.dashboard=true
      - --certificatesresolvers.myresolver.acme.email=${EMAIL}
      - --certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json
      - --certificatesresolvers.myresolver.acme.tlschallenge=true
      #- "--certificatesresolvers.myresolver.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory"
      # uncomment to use staging server, production is default
    ports:
      - "53:53/udp"
      - "53:53"
      - "80:80"
      - "443:443"
      - "853:853"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /finnstats-letsencrypt:/letsencrypt
      - /traefik-logs:/logs
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.dashboard.rule=Host(`${TRAEFIK_DOMAIN}`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))"
      - "traefik.http.routers.dashboard.tls=true"
      - "traefik.http.routers.dashboard.tls.certresolver=myresolver"
      - "traefik.http.routers.dashboard.service=api@internal"

  dns-server:
    container_name: dns-server
    hostname: dns-server
    image: technitium/dns-server:latest
    # Use "host" network mode for DHCP deployments
    #network_mode: "host"
    #ports:
      # - "5380:5380/tcp" #DNS web console
      # - "53:53/udp" #DNS service
      # - "53:53/tcp" #DNS service
      # - "5380:5380/tcp" #Web UI
      # - "5380:5380/tcp" #WEBGUI
      # - "67:67/udp" #DHCP service
      # - "853:853/tcp" #DNS-over-TLS service
      # - "443:443/tcp" #DNS-over-HTTPS service
      # - "80:80/tcp" #DNS-over-HTTPS service certbot certificate renewal
      # - "8053:8053/tcp" #DNS-over-HTTPS using reverse proxy
    environment:
      - DNS_SERVER_DOMAIN=${DOMAIN} #The primary domain name used by this DNS Server to identify itself.
      # - DNS_SERVER_ADMIN_PASSWORD=password #DNS web console admin user password.
      # - DNS_SERVER_ADMIN_PASSWORD_FILE=password.txt #The path to a file that contains a plain text password for the DNS web console admin user.
      # - DNS_SERVER_PREFER_IPV6=false #DNS Server will use IPv6 for querying whenever possible with this option enabled.
      # - DNS_SERVER_OPTIONAL_PROTOCOL_DNS_OVER_HTTP=false #Enables DNS server optional protocol DNS-over-HTTP on TCP port 8053 to be used with a TLS terminating reverse proxy like nginx.
      # - DNS_SERVER_RECURSION=AllowOnlyForPrivateNetworks #Recursion options: Allow, Deny, AllowOnlyForPrivateNetworks, UseSpecifiedNetworks.
      # - DNS_SERVER_RECURSION_DENIED_NETWORKS=1.1.1.0/24 #Comma separated list of IP addresses or network addresses to deny recursion. Valid only for `UseSpecifiedNetworks` recursion option.
      # - DNS_SERVER_RECURSION_ALLOWED_NETWORKS=127.0.0.1, 192.168.1.0/24 #Comma separated list of IP addresses or network addresses to allow recursion. Valid only for `UseSpecifiedNetworks` recursion option.
      # - DNS_SERVER_ENABLE_BLOCKING=false #Sets the DNS server to block domain names using Blocked Zone and Block List Zone.
      # - DNS_SERVER_ALLOW_TXT_BLOCKING_REPORT=false #Specifies if the DNS Server should respond with TXT records containing a blocked domain report for TXT type requests.
      # - DNS_SERVER_FORWARDERS=1.1.1.1, 8.8.8.8 #Comma separated list of forwarder addresses.
      # - DNS_SERVER_FORWARDER_PROTOCOL=Tcp #Forwarder protocol options: Udp, Tcp, Tls, Https, HttpsJson.
    volumes:
      - /technitium/config:/etc/dns/config
    restart: unless-stopped
    labels:
      - traefik.enable=true
      - traefik.http.routers.technitium.rule=Host(`${DOMAIN}`)
      - traefik.http.routers.technitium.entrypoints=websecure
      - traefik.http.routers.technitium.service=technitium-service
      - traefik.http.routers.technitium.tls.certresolver=myresolver
      - traefik.http.routers.technitium.tls=true
      - traefik.http.services.technitium-service.loadbalancer.server.port=5380

      - traefik.tcp.routers.dnstls.rule=HostSNI(`${DOMAIN}`)
      - traefik.tcp.routers.dnstls.service=dnstls-service
      - traefik.tcp.routers.dnstls.entrypoints=dnstls
      - traefik.tcp.routers.dnstls.tls.certresolver=myresolver
      - traefik.tcp.routers.dnstls.tls=true
      - traefik.tcp.services.dnstls-service.loadbalancer.server.port=53

      - traefik.tcp.routers.dnstcp.rule=HostSNI(`*`)
      - traefik.tcp.routers.dnstcp.service=dnstcp-service
      - traefik.tcp.routers.dnstcp.entrypoints=dnstcp
      - traefik.tcp.services.dnstcp-service.loadbalancer.server.port=53

      - traefik.udp.routers.dnsudp.service=dnsudp-service
      - traefik.udp.routers.dnsudp.entrypoints=dnsudp
      - traefik.udp.services.dnsudp-service.loadbalancer.server.port=53
```
