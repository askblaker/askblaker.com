---
title: Docker-compose with ufw-docker in Ubuntu 20.04
date: 2020-11-02T12:00:00+00:00
tags:
- docker
- docker-compose
- ufw-docker
- ufw
---

## Problem

During development using docker-compose on Ubuntu, I realized something weird was happening with my UFW. It just did not work as it should. Publishing ports in docker-compose just overrid any UFW settings, and I was no longer able to use UFW to control the traffic.   

You could say that if you expose a port, the purpose is to keep it open, but sometimes I still want to use a firewall to open/close port access, without disturbing the running services. So I need UFW to work. And because of how Docker and UFW/Ubuntu works, it is not very easy to solve. The current best solution i have found is [ufw-docker](https://github.com/chaifeng/ufw-docker).

## Solution

```bash
sudo wget -O /usr/local/bin/ufw-docker \\  
https://github.com/chaifeng/ufw-docker/raw/master/ufw-docker  
sudo chmod +x /usr/local/bin/ufw-docker  
sudo ufw-docker install  
sudo ufw-docker allow traefik 443  
sudo ufw-docker delete allow traefik 443
```

Note! If you use `docker-compose down` and `docker-compose up`, the ip-address of the container might change, and you need to repeat the process.
