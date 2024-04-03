---
title: "fooocus ollama open-webui docker compose"
date: 2024-04-02T00:00:00+00:00
tags:
  - docker
  - docker compose
  - traefik
  - letsencrypt
  - ai
  - llm
  - ollama
  - open-webui
  - fooocus
  - stable diffusion
---

## Components

[ollama](https://ollama.com/) "Get up and running with large language models. Run Llama 2, Code Llama, and other models. Customize and create your own."  
[open-webui](https://docs.openwebui.com/) "Open WebUI is an extensible, feature-rich, and user-friendly self-hosted WebUI designed to operate entirely offline. It supports various LLM runners, including Ollama and OpenAI-compatible APIs."  
[fooocus](https://github.com/lllyasviel/Fooocus) "Fooocus is an image generating software, similar to stable diffusion with focus on ease of use"  
[labmdalabs](https://lambdalabs.com/) "On-demand & reserved cloud NVIDIA GPUs for AI training & inference. " A host that actually has GPUs available for rapid testing.
[civitai](https://civitai.com/) "Platform for discovering, creating, and sharing generative art."


## Note
A GPU will is highly recommended for increased speed.

## Get started
1. Set a A records for subdomains for your domain pointing to the ip address where you are running this (e.g. webui.ai.yourdomain.com etc..).  
2. Add an .env file and a compose.yml as described below
3. Run `docker compose up -d`  

.env  
```bash
DOMAIN=mydomain.com
EMAIL=me@mydomain.com

# To create appropriately dollar sign escaped string, use below command,
# but replace 'user' with your desired username
# echo $(htpasswd -nB user) | sed -e s/\\$/\\$\\$/g
BASICAUTH=user:$$2y$$05$$RzCTcll/K4PZtxs1668Ulu7MA/HYhx/ADAJB.6OTAq30JFWT9WPXm
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
      # comment out below line to use production server, production is default
      - --certificatesresolvers.myresolver.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /traefik-letsencrypt:/letsencrypt
      - /traefik-logs:/logs
    labels:
      - traefik.enable=true
      - traefik.http.routers.dashboard.rule=Host(`traefik.ai.${DOMAIN}`) && (PathPrefix(`/api`) || PathPrefix(`/dashboard`))
      - traefik.http.routers.dashboard.tls=true
      - traefik.http.routers.dashboard.tls.certresolver=myresolver
      - traefik.http.routers.dashboard.service=api@internal
      - traefik.http.routers.dashboard.middlewares=auth
      - traefik.http.middlewares.auth.basicauth.users=${BASICAUTH}


  ollama:
    volumes:
      - ollama:/root/.ollama
    container_name: ollama
    pull_policy: always
    tty: true
    restart: unless-stopped
    image: ollama/ollama:latest
    # Comment out everything under deploy if you do not have a nvidia GPU
    # (its not required but recommended for speed)
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['0']
              capabilities: [compute, utility]

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    volumes:
      - open-webui:/app/backend/data
    depends_on:
      - ollama
    ports:
      - "8080:8080"
    environment:
      - 'OLLAMA_BASE_URL=http://ollama:11434'
      - 'WEBUI_SECRET_KEY=123'
    extra_hosts:
      - host.docker.internal:host-gateway
    restart: unless-stopped
    labels:
      - traefik.enable=true
      - traefik.http.routers.webui.rule=Host(`webui.ai.${DOMAIN}`)
      - traefik.http.routers.webui.service=webui
      - traefik.http.routers.webui.entrypoints=websecure
      - traefik.http.routers.webui.tls=true
      - traefik.http.routers.webui.tls.certresolver=myresolver
      - traefik.http.services.webui.loadBalancer.server.port=8080
  
  fooocus:
    build: ./Fooocus 
    image: fooocus
    container_name: fooocus
    environment:
      - CMDARGS=--listen    # Arguments for launch.py.
      - DATADIR=/content/data   # Directory which stores models, outputs dir
      - config_path=/content/data/config.txt
      - config_example_path=/content/data/config_modification_tutorial.txt
      - path_checkpoints=/content/data/models/checkpoints/
      - path_loras=/content/data/models/loras/
      - path_embeddings=/content/data/models/embeddings/
      - path_vae_approx=/content/data/models/vae_approx/
      - path_upscale_models=/content/data/models/upscale_models/
      - path_inpaint=/content/data/models/inpaint/
      - path_controlnet=/content/data/models/controlnet/
      - path_clip_vision=/content/data/models/clip_vision/
      - path_fooocus_expansion=/content/data/models/prompt_expansion/fooocus_expansion/
      - path_outputs=/content/app/outputs/    # Warning: If it is not located under '/content/app', you can't see history log!
    volumes:
      - fooocus-data:/content/data
      #- ./models:/import/models   # Once you import files, you don't need to mount again.
      #- ./outputs:/import/outputs  # Once you import files, you don't need to mount again.
    tty: true
    # Comment out everything under deploy if you do not have a nvidia GPU
    # (it is a requirement, so it might not work without it)
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              device_ids: ['0']
              capabilities: [compute, utility]
    labels:
      - traefik.enable=true
      - traefik.http.routers.fooocus.rule=Host(`fooocus.ai.${DOMAIN}`)
      - traefik.http.routers.fooocus.service=fooocus
      - traefik.http.routers.fooocus.entrypoints=websecure
      - traefik.http.routers.fooocus.tls=true
      - traefik.http.routers.fooocus.tls.certresolver=myresolver
      - traefik.http.services.fooocus.loadBalancer.server.port=7865

volumes:
  ollama: {}
  open-webui: {}
  fooocus-data: {}

```
