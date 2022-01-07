# Home Assistant - Installation

## Governing principles

- We want a clean and module-based installation.
  - Store all config/data under `/srv`.
  - Utilize docker:
    - Official docker images onlsudo docky.
    - With volumes for data/config, and only where required bind-mounts.
    - Use `docker cp`, `docker save` or similar to copy/backup data.
- MariaDB as main database-engine.
- InfluxDB as history database-engine.
- Grafana as data visualization-engine.
- Setup a RPI instance with [Raspberry PI install](https://github.com/slittorin/raspberrypi-install/).
  - Preferably it shall have an SSD disk to not degrade/destroy SD card.
  - For now we have all on the same RPI, we may need move InfluxDB and/or Grafana later.
- If not otherwise stated, the user `pi` performs all actions.

## Preparation

1. Under `/srv`:
   - Create the file `.env`.
   - Create the file `/srv/docker-compose.yml` with the following content:
     ```
     version: '3'
     
     services:
     
     volumes:
     ```

## Installation for MariaDB

1. Check versions of available docker images for MariaDB at [Docker - MariaDB](https://hub.docker.com/_/mariadb).
   - If you do not want the latest version, copy the version number.
   - At time of writing the latest version is 10.6 (isolated with `sudo docker image inspect mariadb`).
2. Create the directory `/srv/ha-db`, and the following sub-directories:
   - `var/run/mysqld` - To be able to use sockets (mysqld.sock) that is faster and takes less resources than TCP.
3. For the following file `/srv/.env` add the following content:
```
HA_DB_HOSTNAME=localhost
HA_DB_ROOT_PASSWORD=[not shown here]
HA_DB_DATABASE=ha-db
HA_DB_USER=ha-db-user
HA_DB_PASSWORD=[not shown here]
```
4. For the following file `/srv/docker-compose.yml` add the following content after 'services:' and last added service (keep spaces):
```
# Service: Home Assistant database.
# -----------------------------------------------------------------------------------
  ha-db:
# Add version number if necessary, otherwise keep 'latest'.
    image: mariadb:latest
    container_name: ha-db
# We do not have any ports here as we want bridge-based docker network only within the host.
    network_mode: bridge
    restart: always
    env_file:
      - .env
    environment:
      - MYSQL_ROOT_PASSWORD=${HA_DB_ROOT_PASSWORD}
      - MYSQL_DATABASE=${HA_DB_DATABASE}
      - MYSQL_USER=${HA_DB_USER}
      - MYSQL_PASSWORD=${HA_DB_PASSWORD}
    volumes:
      - "ha-db-data:/var/lib/mysql"
      - "ha-db-config:/etc/mysql/"
      - "/srv/ha-db/var/run/mysqld:/var/run/mysqld"
```
5. For the following file `/srv/docker-compose.yml` add the following content after 'volumes:' and last added volume (keep spaces):
```
  ha-db-data:
  ha-db-config:
```
6. In the `/srv` directory:
   - Pull the docker image first with `sudo docker-compose pull`.
   - Build, create, start, and attach the MariaDB-container with `sudo docker-compose up -d` (dependent on previous state, you may want to add `--force-recreate`. The output should look like the following:
   ```shell
   Creating volume "srv_ha-db-data" with default driver
   Creating volume "srv_ha-db-config" with default driver
   Creating ha-db ... done
   ```
   - Verify that the container is running with `sudo docker ps`. The output should look like the following:
   ```shell
   pi@server1:/srv $ sudo docker ps
   CONTAINER ID   IMAGE            COMMAND                  CREATED         STATUS                  PORTS      NAMES
   5edd4cc80759   mariadb:latest   "docker-entrypoint.s…"   6 seconds ago   Up Less than a second   3306/tcp   ha-db
   ```

## Installation for InfluxDB

1. Check versions of available docker images for InfluxDB at [Docker - InfluxDB](https://hub.docker.com/_/influxdb).
   - If you do not want the latest version, copy the version number.
   - At time of writing the latest version is 2.1.1 (isolated with `sudo docker image inspect influxdb`).
2. Create the directory `/srv/ha-history-db`.
3. For the file `/srv/.env` add the following content:
```
HA_HISTORY_DB_HOSTNAME=localhost
HA_HISTORY_DB_ROOT_USER=influxdb_admin
HA_HISTORY_DB_ROOT_PASSWORD=[not shown here]
HA_HISTORY_DB_ORG=lite
HA_HISTORY_DB_BUCKET=ha
```
4. For the following file `/srv/docker-compose.yml` add the following content after 'services:' and last added service (keep spaces):
```
# Service: Home Assistant history database.
# -----------------------------------------------------------------------------------
  ha-history-db:
# Add version number if necessary, otherwise keep 'latest'.
    image: influxdb:latest
    container_name: ha-history-db
# We do not have any ports here as we want bridge-based docker network only within the host.
    network_mode: bridge
    restart: always
    env_file:
      - .env
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=${HA_HISTORY_DB_ROOT_USER}
      - DOCKER_INFLUXDB_INIT_PASSWORD=${HA_HISTORY_DB_ROOT_PASSWORD}
      - DOCKER_INFLUXDB_INIT_ORG=${HA_HISTORY_DB_ORG}
      - DOCKER_INFLUXDB_INIT_BUCKET=${HA_HISTORY_DB_BUCKET}
    volumes:
      - "ha-history-db-data:/var/lib/influxdb"
      - "ha-history-db-config:/etc/influxdb"
```
5. For the following file `/srv/docker-compose.yml` add the following content after 'volumes:' and last added volume (keep spaces):
```
  ha-history-db-data:
  ha-history-db-config:
```
6. In the `/srv` directory:
   - Pull the docker image first with `sudo docker-compose pull`.
   - Build, create, start, and attach the InfluxDB-container with `sudo docker-compose up -d`. The output should look like the following:
   ```shell
   pi@server1:/srv $ sudo docker-compose up -d
   Creating volume "srv_ha-history-db-data" with default driver
   Creating volume "srv_ha-history-db-config" with default driver
   Recreating ha-db       ... done
   Creating ha-history-db ... done
   ```
   - Verify that the container is running with `sudo docker ps`. The output should look like the following:
   ```shell
   CONTAINER ID   IMAGE             COMMAND                  CREATED          STATUS          PORTS      NAMES
   9fea6e4534e2   mariadb:latest    "docker-entrypoint.s…"   37 seconds ago   Up 35 seconds   3306/tcp   ha-db
   a3c0df2420dc   influxdb:latest   "/entrypoint.sh infl…"   37 seconds ago   Up 36 seconds   8086/tcp   ha-history-db
   ```

## Installation for Grafana

1. Check versions of available docker images for InfluxDB at [Docker - InfluxDB](https://hub.docker.com/_/influxdb).
   - If you do not want the latest version, copy the version number.
   - At time of writing the latest version is 2.1.1 (isolated with `sudo docker image inspect influxdb`).
2. Create the directory `/srv/ha-history-db`.
3. For the file `/srv/.env` add the following content:
```
HA_HISTORY_DB_HOSTNAME=localhost
HA_HISTORY_DB_ROOT_USER=influxdb_admin
HA_HISTORY_DB_ROOT_PASSWORD=[not shown here]
HA_HISTORY_DB_ORG=lite
HA_HISTORY_DB_BUCKET=ha
```
