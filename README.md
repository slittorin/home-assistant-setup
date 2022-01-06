# Home Assistant - Installation

## Prequisities

- We want a clean and module-based installation.
  - Store under `/srv`.
  - Utilize Bind mounts in docker to store data outside the containers.
- We utilize official docker images only.
- We utilize MariaDB as main database-engine.
- We utilize InfluxDB as history database-engine.
- Setup a RPI instance with [Raspberry PI install](https://github.com/slittorin/raspberrypi-install/).
  - Preferably it shall have an SSD disk to not degrade/destroy SD card.
  - For now we have all on the same RPI, we may need move InfluxDB and/or Grafana later.

## Installation for MariaDB

1. Check versions of available docker images for MariaDB at [Docker - MariaDB](https://hub.docker.com/_/mariadb).
   - If you do not want the latest version, copy the version number.
2. Create the directory `/srv/ha-db`, and the following sub-directories:
   - `var/lib/mysql` - To access the data (/var/lib/mysql).
   - `var/run/mysqld` - To be able to use sockets (mysqld.sock) that is much faster and takes less resources than TCP.
3. Create the following file `/srv/.env` with the following content:
```
HA_DB_HOSTNAME=localhost
HA_DB_ROOT_PASSWORD=[not shown here]
HA_DB_DATABASE=ha-db
HA_DB_USER=ha-db-user
HA_DB_PASSWORD=[not shown here]
```
4. Create the following file `/srv/docker-compose.yml` with the following content:
```
version: '3'

services:
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
    volumes:cd ..
      - "/srv/ha-db/var/lib/mysql:/var/lib/mysql"
      - "/srv/ha-db/var/run/mysqld:/var/run/mysqld"
```
3. In the `/srv` directory:
   - Pull the docker image first with `sudo docker-compose pull`.
   - Build, create, start, and attach the MariaDB-container with `sudo docker-compose up -d` (dependent on previous state, you may want to add `--force-recreate`. The output should look like the following:
   ```shell
   pi@server1:/srv $ sudo docker-compose up -d
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
2. Create the directory `/srv/ha-history-db`, and the following sub-directory:
   - `var/lib/influxdb` - To access the data (/var/lib/influxdb).
   - 'etc/influxdb` - To access the config file.
3. For the file `/srv/.env` add the following content:
```
HA_HISTORY_DB_HOSTNAME=localhost
HA_HISTORY_DB_ROOT_USER=influxdb_admin
HA_HISTORY_DB_ROOT_PASSWORD=[not shown here]
HA_HISTORY_DB_ORG=lite
HA_HISTORY_DB_BUCKET=ha
```
6. For the following file `/srv/docker-compose.yml` add the following content:
```

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
      - "/srv/ha-history-db/var/lib/influxdb:/var/lib/influxdb"
      - "/srv/ha-history-db/etc/influxdb:/etc/influxdb"
```
3. In the `/srv` directory:
   - Pull the docker image first with `sudo docker-compose pull`.
   - Build, create, start, and attach the InfluxDB-container with `sudo docker-compose up -d`. The output should look like the following:
   ```shell
   ```
   - Verify that the container is running with `sudo docker ps`. The output should look like the following:
   ```shell
   ```

## Installation for Grafana

Coming soon.
