---
published: true
title: Install OwnCloud with Docker

author: Jevy 
category: Gist
tags: [Geek]
date: 2019-06-07
---

I struggled to install OwnCloud directly on Ubuntu 18. I spent hours installing and configuring its dependencies but failed at the database script execution step.

While searching for a solution, I found the Dockerized OwnCloud and discovered it is pretty easy to set up. 

Here is the installation/configuration log:

**Install docker and docker-compose**
- [docker](https://docs.docker.com/install/linux/docker-ce/ubuntu/)
- [docker-compose](https://docs.docker.com/compose/install/)

**Prepare a docker-compose.yml** :

I referenced the [official Docker installation guide](https://doc.owncloud.com/server/admin_manual/installation/docker/)
and this [example docker-compose.yml](https://raw.githubusercontent.com/owncloud-docker/server/master/docker-compose.yml) to create my own `docker-compose.yml`.

Because I have an RDS instance on the cloud and NAS storage mounted on this server, I wanted to reuse them. So I removed the volumes and db dependencies from the yml file.
Because this OwnCloud file sharing will be used by very few users, and the VM has extra charges for CPU usage, I removed redis. 

```yml
version: '2.1'

services:
  owncloud:
    image: owncloud/server:10.0
    restart: always
    ports:
      - 8080:8080
    environment:
      - OWNCLOUD_DOMAIN=localhost
      - OWNCLOUD_DB_TYPE=mysql
      - OWNCLOUD_DB_NAME=db_name
      - OWNCLOUD_DB_USERNAME=db_username
      - OWNCLOUD_DB_PASSWORD=db_password
      # db can be any remote url with accessing privilege
      - OWNCLOUD_DB_HOST=db_url
      - OWNCLOUD_ADMIN_USERNAME=admin
      - OWNCLOUD_ADMIN_PASSWORD=admin
      # for owncloud >= 9.0, add this config to allow local directory as external storage
      - OWNCLOUD_ALLOW_EXTERNAL_LOCAL_STORAGE=true
    healthcheck:
      test: ["CMD", "/usr/bin/healthcheck"]
      interval: 30s
      timeout: 10s
      retries: 5
    volumes:
      - /mnt/data/owncloud:/mnt/data
      - /other-directory1:/mnt/data/d1
      - /other-directory2:/mnt/data/d2

```
Use command `sudo docker-compose up -d` to start owncloud, and access it through `server_ip:8080`. 
Please add port `8080` to your VM's network route rule. 

**Add local storage** :

Reference this [Local storage](https://doc.owncloud.com/server/admin_manual/configuration/files/external_storage/local.html) to add local directory. 


**Useful notes**
- scan files copied to owncloud directory: `docker-compose exec owncloud occ files:scan --all`