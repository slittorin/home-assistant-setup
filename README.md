# Home Assistant - Installation

## Prequisities

- We want a clean and module-based installation.
- We utilize official docker images only.
- We utilize MariaDB as main database-engine.
- We utilize InfluxDB as history database-engine.
- Setup a RPI instance with [Raspberry PI install](https://github.com/slittorin/raspberrypi-install/).

## Installation for MariaDB

1. Check versions of available docker images for MariaDB at [Docker - MariaDB](https://hub.docker.com/_/mariadb).
   - If you do not want the latest version, copy the version number.
4. Create the directory `/srv/ha-db`, and the following sub-directories:
   - `lib` - To capture the data (/var/lib/mysql).
   - `log` - To capture the logs (/var/log/mysql). Are these really needed?
   - `etc` - To capture the config (/etc/mysql). Are these really needed?
6. Create the following file `/srv/docker-compose.yml` with the following content:
```
version: '3'

services:
  ha-db:
# Add version number if necessary, otherwise keep 'latest'.
    image: mariadb:latest
    container_name: ha-db
# We do not have any ports here as we want bridge-based docker network only within the host.
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD: [not shown here]
      - MYSQL_DATABASE: ha-db
      - MYSQL_USER: ha-db-user
      - MYSQL_PASSWORD: [not shown here]
    volumes:
# We utilize Bind mounts.
      - "/srv/ha-db/lib:/var/lib/mysql"
# Are these two needed?
      - "/srv/ha-db/log:/var/log/mysql"
      - "/srv/ha-db/etc:/etc/mysql"
```
3. In the `/srv` directory:
   - Pull the docker image first with `sudo docker-compose pull`.
   - Build, create, start, and attach the MariaDB-container with `sudo docker-compose up -d`. The output should look like the following:
   ```shell
   pi@server1:/srv $ sudo docker-compose up -d
   Creating network "srv_default" with the default driver
   Creating srv_ha-db_1 ... done
   ```
   - Verify that the container is running with `sudo docker ps`. The output should look like the following:
   ```shell
   pi@server1:/srv $ sudo docker ps
   CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS      NAMES
   5aea58eeea44   mariadb   "docker-entrypoint.s…"   3 minutes ago   Up 3 minutes   3306/tcp   srv_ha-db_1
   ```

## Installation for InfluxDB

1. Check versions of available docker images for MariaDB at [Docker - InfluxDB](https://hub.docker.com/_/influxdb).
   - If you do not want the latest version, copy the version number.
4. Create the directory `/srv/ha-history-db`, and the following sub-directory:
   - `lib` - To capture the data (/var/lib/influxdb).
6. For the following file `/srv/docker-compose.yml` add the following content:
```

ha-history-db:
# Add version number if necessary, otherwise keep 'latest'.
    image: influxdb:latest
    container_name: ha-history-db
# We do not have any ports here as we want bridge-based docker network only within the host.
    restart: always
    environment:
      - INFLUXDB_DB=db0
      - INFLUXDB_USER=telegraf
      - INFLUXDB_ADMIN_ENABLED=true
      - INFLUXDB_ADMIN_USER=admin
      - INFLUXDB_ADMIN_PASSWORD=Welcome1 
    volumes:
# We utilize Bind mounts.
      - "/srv/ha-history-db/lib:/var/lib/influxdb"
      - "/srv/ha-history-db/influxdb.conf:/etc/influxdb/influxdb.conf:ro"
      - "/srv/ha-history-db/init:/docker-entrypoint-initdb.d"
```
3. In the `/srv` directory:
   - Pull the docker image first with `sudo docker-compose pull`.
   - Build, create, start, and attach the InfluxDB-container with `sudo docker-compose up -d`. The output should look like the following:
   ```shell
   ```
   - Verify that the container is running with `sudo docker ps`. The output should look like the following:
   ```shell
   ```
