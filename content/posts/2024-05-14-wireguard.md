---
title: "Exposing homelab services publicly"
date: 2024-05-14T00:00:00+00:00
tags:
  - wireguard
  - caddy
  - letsencrypt
  - bastion server
  - docker
  - homelab
---

## Components

[wireguard](https://www.wireguard.com/) "WireGuard® is an extremely simple yet fast and modern VPN that utilizes state-of-the-art cryptography. It aims to be faster, simpler, leaner, and more useful than IPsec, while avoiding the massive headache. "  

[caddy](https://caddyserver.com/) "
'The Ultimate Server' - makes your sites more secure, more reliable, and more scalable than any other solution."  

[docker](https://www.docker.com/) "Accelerate how you build, share, and run applications. Docker helps developers build, share, run, and verify applications anywhere — without tedious environment configuration or management."  


## Introduction
Lets say you have a service running somewhere, like in a homelab, that you would like to make available on the internet. But you would not like to make the network where this service is running available. You would also like to own all of these services yourself. (If not you could probably get away with using something like cloudflare tunnels).

So what this guide will help you do, is:
1. Run a service
2. Run a load balancer / reverse proxy on a cloud VPS
3. Connect the service and reverse proxy via an encrypted connection
4. Add automatic renewing https
5. Add rate limiting

## Guide
### 1. Get a VPS and a domain
You need a cloud machine with a publicly available ip address, and a domain where you can edit the dns records.

### 2. A records
Set a A records for subdomains for your domain pointing to the ip address where you are running this (e.g. homelab.yourdomain.com etc..).  

### 3. Ping your server ip
```bash
ping 12.123.123.123
PING 12.123.123.123 (12.123.123.123) 56(84) bytes of data.
64 bytes from 12.123.123.123: icmp_seq=1 ttl=47 time=26.5 ms
64 bytes from 12.123.123.123: icmp_seq=2 ttl=47 time=36.0 ms
64 bytes from 12.123.123.123: icmp_seq=3 ttl=47 time=81.3 ms
64 bytes from 12.123.123.123: icmp_seq=4 ttl=47 time=101 ms
```

### 4. Ping your server domain name
```bash
ping test.yourdomain.com
PING test.yourdomain.com (12.123.123.123) 56(84) bytes of data.
64 bytes from ns.yourprovider.com (12.123.123.123): icmp_seq=1 ttl=47 time=102 ms
64 bytes from ns.yourprovider.com (12.123.123.123): icmp_seq=2 ttl=47 time=124 ms
64 bytes from ns.yourprovider.com (12.123.123.123): icmp_seq=3 ttl=47 time=45.0 ms
64 bytes from ns.yourprovider.com (12.123.123.123): icmp_seq=4 ttl=47 time=63.1 ms
```

### 5. Run caddy as a test
On server:
```bash
docker run --rm -p 2015:2015 caddy /bin/sh -c 'caddy respond --listen :2015 "Hello World!"'
```

On any client:
On server:
```bash
curl <server ip>:2015; echo
Hello world!
```

### 6. Run a test service on your protected machine
```bash
sudo docker run --rm -p 41414:80 nginxdemos/hello
```

### 7. Check that it is running
On the protected machine:  
```bash
curl localhost:41414
<A WALL OF HTML SHOULD APPEAR HERE>
```

### 8. Install wireguard on both cloud vps and protected server
```bash
apt update 
apt install wireguard
```

In a suitable folder:
```bash
wg genkey | tee privatekey | wg pubkey > publickey
chmod 600 privatekey publickey
```

```
# Server
# /etc/wireguard/wg0.conf
[Interface]
Address = 10.10.0.1/32
PrivateKey = <server's privatekey>
ListenPort = 51820

[Peer]
PublicKey = <client's publickey>
AllowedIPs = 192.168.2.2/32
```


```
# Client
# /etc/wireguard/wg0.conf
[Interface]
Address = 10.10.10.2/32
PrivateKey = <client's privatekey>

[Peer]
PublicKey = <server publickey>
Endpoint = <servers ip>:51820
AllowedIPs = 192.168.2.0/32
PersistentKeepalive = 25
```

To start and stop
```bash
wg-quick start wg0
wg-quick stop wg0
```

To add it as a systemd service (auto restart)
```bash
sudo systemctl enable wg-quick@wg0.service
```

To start or stop the service
```bash
sudo systemctl start wg-quick@wg0.service
sudo systemctl stop wg-quick@wg0.service
```
### 9. Check wireguard connectivity
```bash
# From protected server
ping 10.10.0.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=37.0 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=27.2 ms
```

```bash
# From vps
ping 10.10.0.2
PING 10.10.10.2 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.2: icmp_seq=1 ttl=64 time=26.9 ms
64 bytes from 10.10.10.2: icmp_seq=2 ttl=64 time=27.3 ms
```

```bash
# From vps
# (Make sure the hello world service is running on the protected server)
curl 10.10.0.2:41414
<Wall of html>
```

### 10. Point caddy to the protected service
```bash
docker run --rm -p 2015:2015 caddy /bin/sh -c \
'caddy reverse-proxy --from :2015 --to  10.10.10.2:41414'
#with a web browser, access http://<vps ip>:2015
```

### 11. Add https
```bash
docker run --rm -p 80:80 -p 443:443 caddy /bin/sh -c \
'caddy reverse-proxy --from test.yourdomain.com --to 10.10.10.2:41414'
#with a web browser, access https://test.yourdomain.com
```

### 12. Add rate limiting
As an example of using caddy plug in, dockerfile and caddyfile

1. Add Dockerfile
```Dockerfile
FROM caddy:builder-alpine AS builder

RUN xcaddy build \
	--with github.com/mholt/caddy-ratelimit

FROM caddy:latest

COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

2. Build with the new Dockerfile
```bash
docker build . -t caddy 
```

3. Add Caddyfile
```Caddyfile
{
	order rate_limit before reverse_proxy
}

test.askblaker.com {
	rate_limit {
		zone dynamic_zone {
			key {http.request.remote_ip}
			events 10
			window 5s
		}
	}

	reverse_proxy 10.10.10.2:41414
}
```

4. Run caddy with mounted caddyfile, and a volume to store certs (and to persist them between restarts)
```bash
docker run -d --restart unless-stopped --name caddy -p 80:80 -p 443:443 \ 
-v ${PWD}/caddy/Caddyfile:/etc/caddy/Caddyfile -v ${PWD}/caddy_data:/data caddy
```

5. To update changes in the Caddfile you can either just restart the container
```bash
docker restart caddy
```
or reload the configuration without restarting
```bash
docker exec -it caddy /bin/sh -c 'caddy reload --config /etc/caddy/Caddyfile'
```
