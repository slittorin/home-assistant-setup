# Home Assistant - Installation

## Governing principles

- We want to utilize RPI standard OS, and not HASS.io that would get the whole server to be only for Home Assistant.
  - We want to be more in control and utilize the server for other tasks also.
- We want a clean and module-based installation.
  - Store all config/data under `/srv`.
  - Utilize docker:
    - Official docker images onlsudo docky.
    - With volumes for data/config, and only where required bind-mounts.
    - Use `docker cp`, `docker save` or similar to copy/backup data.
- MariaDB as main database-engine.
  - We utiize mysqld-volume for mysqld-socket communication that is faster and takes less resources than TCP.
- InfluxDB as history database-engine.
- Grafana as data visualization-engine.
- Home Assistant as automation-engine.
  - We utilize the mysqld-socket volume.
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
   - If you do not want the 'latest' version, , use version number.
   - At time of writing (20220207) the 'latest' version is 10.6.5 (isolated with `sudo docker image inspect mariadb` and looking for 'MARIADB_VERSION').
2. Create the directory `/srv/ha-db`, and the following sub-directories:
   - At present no specific directories are used.
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
    restart: on-failure
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
      - "ha-db-mysqld:/var/run/mysqld"
```
5. For the following file `/srv/docker-compose.yml` add the following content after 'volumes:' and last added volume (keep spaces):
```
  ha-db-data:
  ha-db-config:
  ha-db-mysqld:
```
6. In the `/srv` directory:
   - Pull the docker image first with `sudo docker-compose pull`.
   - Build, create, start, and attach the MariaDB-container with `sudo docker-compose up -d` (dependent on previous state, you may want to add `--force-recreate`. The output should look like the following:
   ```shell
   Creating volume "srv_ha-db-data" with default driver
   Creating volume "srv_ha-db-config" with default driver
   Creating volume "srv_ha-db-mysqld" with default driver
   Creating ha-db ... done
   ```
   - Verify that the container is running with `sudo docker ps`. The output should look like the following:
   ```shell
   CONTAINER ID   IMAGE            COMMAND                  CREATED         STATUS                  PORTS      NAMES
   5edd4cc80759   mariadb:latest   "docker-entrypoint.s…"   6 seconds ago   Up Less than a second   3306/tcp   ha-db
   ```

## Installation for InfluxDB

1. Check versions of available docker images for InfluxDB at [Docker - InfluxDB](https://hub.docker.com/_/influxdb).
   - If you do not want the 'latest' version, use version number.
   - At time of writing (20220207) the 'latest' version is 2.1.1 (isolated with `sudo docker image inspect influxdb` and looking for 'INFLUXDB_VERSION').
2. Create the directory `/srv/ha-history-db`, and the following sub-directories:
   - At present no specific directories are used.
4. For the file `/srv/.env` add the following content:
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
    restart: on-failure
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
   Creating volume "srv_ha-history-db-data" with default driver
   Creating volume "srv_ha-history-db-config" with default driver
   ha-db is up-to-date
   Creating ha-history-db ... done
   ```
   - Verify that the container is running with `sudo docker ps`. The output should look like the following:
   ```shell
   CONTAINER ID   IMAGE             COMMAND                  CREATED          STATUS          PORTS      NAMES
   9fea6e4534e2   mariadb:latest    "docker-entrypoint.s…"   37 seconds ago   Up 35 seconds   3306/tcp   ha-db
   a3c0df2420dc   influxdb:latest   "/entrypoint.sh infl…"   37 seconds ago   Up 36 seconds   8086/tcp   ha-history-db
   ```

## Installation for Grafana

1. Check versions of available docker images for Grafana at [Docker - Grafana](https://hub.docker.com/r/grafana/grafana).
   - If you do not want the 'latest' version, use version number, or use 'main'.
   - At time of writing (20220207) the 'latest' version is 8.3.3 (isolated with running the command `/usr/share/grafana/bin/grafana-server -v` on the container).
2. Create the directory `/srv/ha-grafana`, and the following sub-directories:
   - At present no specific directories are used.
4. For the file `/srv/.env` add the following content:
```
HA_GRAFANA_HOSTNAME=localhost
```
4. For the following file `/srv/docker-compose.yml` add the following content after 'services:' and last added service (keep spaces):
```
# Service: Home Assistant grafana.
# -----------------------------------------------------------------------------------
  ha-grafana:
# Add version number if necessary, otherwise keep 'latest'.
    image: grafana/grafana:latest
    container_name: ha-grafana
# We do not have any ports here as we want bridge-based docker network only within the host.
    network_mode: bridge
    restart: on-failure
    env_file:
      - .env
    environment:
# This will allow you to access your Grafana dashboards without having to log in and disables a security measure that prevents you from using Grafana in an iframe.
      - GF_AUTH_DISABLE_LOGIN_FORM=true
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
      - GF_SECURITY_ALLOW_EMBEDDING=true
    volumes:
      - "ha-grafana-data:/var/lib/grafana"
      - "ha-grafana-config:/etc/grafana"
```
5. For the following file `/srv/docker-compose.yml` add the following content after 'volumes:' and last added volume (keep spaces):
```
  ha-grafana-data:
  ha-grafana-config:
```
6. In the `/srv` directory:
   - Pull the docker image first with `sudo docker-compose pull`.
   - Build, create, start, and attach the InfluxDB-container with `sudo docker-compose up -d`. The output should look like the following:
   ```shell
   Creating volume "srv_ha-grafana-data" with default driver
   Creating volume "srv_ha-grafana-config" with default driver
   ha-history-db is up-to-date
   ha-db is up-to-date
   Creating ha-grafana ... done
   ```
   - Verify that the container is running with `sudo docker ps`. The output should look like the following:
   ```shell
   CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS      NAMES
   d2030bfce42a   mariadb:latest           "docker-entrypoint.s…"   22 seconds ago   Up 20 seconds   3306/tcp   ha-db
   3e82d3d00e9f   influxdb:latest          "/entrypoint.sh infl…"   23 seconds ago   Up 21 seconds   8086/tcp   ha-history-db
   aae462b4ae93   grafana/grafana:latest   "/run.sh"                23 seconds ago   Up 21 seconds   3000/tcp   ha-grafana
   ```

## Installation for Home Assistant

To be able to utilize most features of HA, such as Add-Ons, we install HA Supervised method instead.
The below is derived from [Install Home Assistant Supervised](https://peyanski.com/how-to-install-home-assistant-supervised-official-way/).
HA will state that we are running an unsupported installation.

1. Add necessary packages with `sudo apt-get install jq wget curl avahi-daemon udisks2 libglib2.0-bin network-manager dbus apparmor -y`.
2. Create the directory `/srv/ha`, and the following sub-directories:
   - `downloads'. To store downloaded packages.
3. Isolate the latest HA Agent from [Home Assistant Agent - Download](https://github.com/home-assistant/os-agent/releases/latest).
   - At 20220207 it is 1.2.2.
4. Go to directory '/srv/ha/downloads' and download the right HA Agent package with 'sudo wget https://github.com/home-assistant/os-agent/releases/download/1.2.2/os-agent_1.2.2_linux_aarch64.deb' (change the version according step 3).
   - As we are using 64 bit version for RPI 4, we need to download this version, not armv7 that would otherwise work for 32 bit version.

# Deprecated - Not valid

## Installation for Home Assistant - Deprecated as of 20220207 - We cannot use all features we need with this type of installation.

1. Check versions of available docker images for Home Assistant at [Docker - Home Assistant](https://hub.docker.com/r/homeassistant/home-assistant).
   - If you do not want the 'latest' version, use version number, or use 'stable'.
   - At time of writing (20220207) the 'stable' version is 2021.12.8 (isolated with `sudo docker image inspect homeassistant/home-assistant:stable` and looking for 'org.opencontainers.image.version').
2. Create the directory `/srv/ha`, and the following sub-directories:
   - `config` - To easily get access to the configuration files for HA.
4. For the following file `/srv/docker-compose.yml` add the following content after 'services:' and last added service (keep spaces):
```
# Service: Home Assistant.
# -----------------------------------------------------------------------------------
  ha:
# We want to have the latest stable version.
    image: homeassistant/home-assistant:stable
    container_name: ha
# We want supervised mode.
    privileged: true
# We want HA to be accessible on the network.
    ports:
      - "8123:8123"
    restart: on-failure
    env_file:
      - .env
    depends_on:
      - ha-db
      - ha-history-db
      - ha-grafana
    volumes:
      - "/srv/ha/config:/config"
      - "/etc/localtime:/etc/localtime:ro"
      - "ha-db-mysqld:/var/run/mysqld"
```
4. In the `/srv` directory:
   - Pull the docker image first with `sudo docker-compose pull`.
   - Build, create, start, and attach the InfluxDB-container with `sudo docker-compose up -d`. The output should look like the following:
   ```shell
   Creating network "srv_default" with the default driver
   ha-db is up-to-date
   ha-grafana is up-to-date
   ha-history-db is up-to-date
   Creating ha ... done
   ```
   - Verify that the container is running with `sudo docker ps`. The output should look like the following:
   ```shell
   CONTAINER ID   IMAGE                                 COMMAND                  CREATED          STATUS          PORTS                    NAMES
   f994d95d319e   homeassistant/home-assistant:stable   "/init"                  24 seconds ago   Up 22 seconds   0.0.0.0:8123->8123/tcp   ha
   e61bdcdb9531   grafana/grafana:latest                "/run.sh"                4 minutes ago    Up 4 minutes    3000/tcp                 ha-grafana
   1be625a981fc   influxdb:latest                       "/entrypoint.sh infl…"   5 minutes ago    Up 5 minutes    8086/tcp                 ha-history-db
   e4117f81816e   mariadb:latest                        "docker-entrypoint.s…"   7 minutes ago    Up 7 minutes    3306/tcp                 ha-db
   ```
