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

