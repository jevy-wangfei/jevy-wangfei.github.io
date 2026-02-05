---

title: Deploying OwnCloud with Docker

category: Gist
tags: [Docker, OwnCloud, DevOps, Ubuntu]
date: 2019-06-07
---

Installing OwnCloud directly on Ubuntu 18 can be challenging due to dependency conflicts and database configuration issues. After struggling with a manual installation, I discovered that the Dockerized version of OwnCloud is much easier to set up and maintain.

Here is my installation and configuration log.

### Prerequisites

- [Install Docker](https://docs.docker.com/install/linux/docker-ce/ubuntu/)
- [Install Docker Compose](https://docs.docker.com/compose/install/)

### Docker Compose Configuration

I referenced the [official OwnCloud Docker installation guide](https://doc.owncloud.com/server/admin_manual/installation/docker/) and the [official example](https://raw.githubusercontent.com/owncloud-docker/server/master/docker-compose.yml) to create a customized `docker-compose.yml`.

**Customizations:**
- **External Database**: I removed the database container dependencies because I wanted to connect to an existing AWS RDS instance.
- **External Storage**: I configured volumes to mount my existing NAS storage.
- **Redis Removed**: Since this instance serves very few users and I wanted to minimize CPU usage on the VM, I removed the Redis service.

Here is the `docker-compose.yml` file:

```yaml
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

### Running OwnCloud

1.  Start the service:
    ```bash
    sudo docker-compose up -d
    ```
2.  Access OwnCloud at `http://server_ip:8080`.
3.  **Note**: Ensure port `8080` is open in your VM's firewall/security group settings.

### Adding Local Storage

To add local directories as external storage, verify that `OWNCLOUD_ALLOW_EXTERNAL_LOCAL_STORAGE=true` is set, and refer to the [Local Storage Configuration](https://doc.owncloud.com/server/admin_manual/configuration/files/external_storage/local.html) documentation.

### Useful Commands

- Scan files manually copied to the data directory:
    ```bash
    docker-compose exec owncloud occ files:scan --all
    ```