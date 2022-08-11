---
title: My Traefik config with let's encrypt wildcards
date: 2022-08-10 12:00:00 -500
categories: [homelab,traefik,let's encrypt]
layout: post
tags: [reverse proxy, let's encrypt] 
---
# Install ansible and docker compose
On the control host install ansible

```bash
 sudo apt install ansible docker-compose -y
 ```

# Run Playbook
```yaml
---
- hosts: all
  become: true
  tasks:
    - name: install pre-reqs
      apt:
        name: "{{item}}"
        state: present
      with_items:
        - "ca-certificates"
        - "curl"
        - "software-properties-common"
        - "docker-compose"
    
    - name: docker repo key
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present
    
    - name: docker repo install
      apt_repository:
        repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable
        state: present
    
    - name: install docker
      apt:
        name: docker-ce
        state: present
        update_cache: yes
    
    - name: start service
      service:
        name: docker
        state: started
        enabled: true

- name: ensure docker-py is installed
  apt:
    name: "{{ item }}"
    state: present
  with_items:
    # - "python-docker"
    - "python3-docker"

- name: add user to docker group
  user:
    name: ryan
    group: docker
    append: yes
```

Make data directory for traefik
```console

mkdir /data/traefik_data
chown -R user:user
```

Create the `acme.json` file
```console
cd /data/traefik_data
touch acme.json
chmod 600 acme.json
```
Create `traefik.yml`

```console
touch traefik.yml
```
Copy config
```yaml
api:
  dashboard: true
  debug: true
entryPoints:
  http:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: https
          scheme: https
  https:
    address: ":443"
serversTransport:
  insecureSkipVerify: true
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
  file:
    filename: /config.yml
certificatesResolvers:
  cloudflare:
    acme:
      email: EMAIL # the cloudflare account email
      storage: acme.json
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "1.0.0.1:53"
```
Create `config.yml`

```yaml
http:
 #region routers 
  routers:
    proxmox:
      entryPoints:
        - "https"
      rule: "Host(`hype.example.net`)"
      middlewares:
        - default-headers
        - middlewares-https-redirectscheme
      tls: {}
      service: proxmox
    pihole:
      entryPoints:
        - "https"
      rule: "Host(`dns.example.net`)"
      middlewares:
        - default-headers
        - addprefix-pihole
        - https-redirectscheme
      tls: {}
      service: pihole
    homeassistant:
      # For Homeassistant config, check: https://www.home-assistant.io/integrations/http/#reverse-proxies
      # This relies on Homeassistant using http. No certs are needed in the Homeassistant config.
      entryPoints:
        - "https"
      rule: "Host(`ha.example.net`)"
      middlewares:
        - default-headers
        - https-redirectscheme
      tls: {}
      service: homeassistant
    truenas:
      entryPoints:
        - "https"
      rule: "Host(`sandrock.example.net`)"
      middlewares:
        - default-headers
        - https-redirectscheme
      tls: {}
      service: truenas
    plex:
      entryPoints:
        - "https"
      rule: "Host(`plex.example.net`)"
      middlewares:
        - default-headers
        - https-redirectscheme
      tls: {}
      service: plex
   
    opnsense:
      entryPoints:
        - "https"fw.example.net`)"
      middlewares:
        - default-headers
        - https-redirectscheme
      tls: {}
      service: opnsense
    pterodactyl:
      entryPoints:
        - "https"
      rule: "Host(`game-server.example.net`)"
      middlewares:
        - default-headers
        - https-redirectscheme
      tls: {}
      service: pterodactyl

#endregion
#region services
  services:
    proxmox:
      loadBalancer:
        servers:
          - url: "https://10.0.3.195:8006"
        passHostHeader: true
    pihole:
      loadBalancer:
        servers:
          - url: "http://10.0.3.253:80"
          - url: "http://10.0.3.251:80"
        passHostHeader: true
    homeassistant:
      loadBalancer:
        servers:
          - url: "http://10.0.3.250:8123"
        passHostHeader: true
    truenas:
      loadBalancer:
        servers:
          - url: "http://10.0.3.194"
        passHostHeader: true
    plex:
      loadBalancer:
        servers:
          - url: "http:10.0.3.190:32400"
        passHostHeader: true
    opnsense:
      loadBalancer:
        servers:
          - url: "https://10.0.2.1"
        passHostHeader: true
    #pterodactyl:
    #  loadBalancer:
    #    servers:
    #      - url: "http://192.168.0.110:80"
    #    passHostHeader: true
#endregion
  middlewares:
    addprefix-pihole:
      addPrefix:
        prefix: "/admin"
    middlewares-https-redirectscheme:
      redirectScheme:
        scheme: https
        permanent: true

    default-headers:
      headers:
        frameDeny: true
        browserXssFilter: true
        contentTypeNosniff: true
        forceSTSHeader: true
        stsIncludeSubdomains: true
        stsPreload: true
        stsSeconds: 15552000
        customFrameOptionsValue: SAMEORIGIN
        customRequestHeaders:
          X-Forwarded-Proto: https


    default-whitelist:
      ipWhiteList:
        sourceRange:
        - "10.0.0.0/8"
        - "192.168.0.0/16"
        - "172.16.0.0/12"

    secured:
      chain:
        middlewares:
        - default-whitelist
        - default-headers
```
Finally Copy the `docker-compose.yml`
```console
vi docker-compose.yml
```
```yaml
version: '3'

services:
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - proxy
    ports:
      - 80:80
      - 443:443
    environment:
      - CF_API_EMAIL=CF_EMAIL
      - CF_DNS_API_TOKEN=TF TOKEN
  
      # be sure to use the correct one depending on if you are using a token or key
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /data/traefik_data/traefik.yml:/traefik.yml:ro
      - /data/traefik_data/acme.json:/acme.json
      - /data/traefik_data/config.yml:/config.yml:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.entrypoints=http"
      - "traefik.http.routers.traefik.rule=Host(`traefik-dash.example.com`)"
      #- "traefik.http.middlewares.traefik-auth.basicauth.users=user:password hash"
      - "traefik.http.middlewares.traefik-https-redirect.redirectscheme.scheme=https"
      - "traefik.http.middlewares.sslheader.headers.customrequestheaders.X-Forwarded-Proto=https"
      - "traefik.http.routers.traefik.middlewares=traefik-https-redirect"
      - "traefik.http.routers.traefik-secure.entrypoints=https"
      - "traefik.http.routers.traefik-secure.rule=Host(`traefik-dash.example.com`)"
      #- "traefik.http.routers.traefik-secure.middlewares=traefik-auth"
      - "traefik.http.routers.traefik-secure.tls=true"
      - "traefik.http.routers.traefik-secure.tls.certresolver=cloudflare"
      - "traefik.http.routers.traefik-secure.tls.domains[0].main=example.com"
      - "traefik.http.routers.traefik-secure.tls.domains[0].sans=*.example.com"
      - "traefik.http.routers.traefik-secure.service=api@internal"
      

networks:
  proxy:
    external: true
```