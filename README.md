# Home Assistant - Setup

Although it would be possible to install HA Supervised on one RPI with all features, it would not be standard and would require alot of tweaks, and I want to keep the setup as standard and future proof as possible (we tried for instance [Install Home Assistant Supervised on RPI](https://peyanski.com/how-to-install-home-assistant-supervised-official-way/) on a single server).

We could also utilize one server with HASS.io and addons for MariaDB, InfluxDB and Grafana, but that would most likely not be future proof as the load will increase as more data is gathered.

Therefore we have gone for a two-server setup according to below.

# Table of content

- [Conceptual design](https://github.com/slittorin/home-assistant-setup#conceptual-design)
- [


## Conceptual design

Instead of one RPI server we will have two:
- homeassistant on VLAN-Server (192.168.3.20):
  - RPI 4 with 250GB SSD disk. Standard HASS.io install, with the following addons:
    - MariaDB.
    - Setup a RHASS.io with [HASS.io install](https://github.com/slittorin/hass-io-install).
- server1 on VLAN-Server (192.168.3.30):
   - RPI 4 with 500GB SSD disk. Standard RPI, with docker and the following services:
     - InfluxDB.
     - Grafana.
   - Intended also to be utilized for other projects.
   - Setup a RPI instance with [Raspberry PI install](https://github.com/slittorin/raspberrypi-install/).


## Governing principles

- Limit the number of exposed ports/services on the Home Assistant.

# Setup for - server 1

## Preparation

1. Under `/srv`:
   - Create the file `.env`.
   - Create the file `/srv/docker-compose.yml` with the following content:
     ```
     version: '3'
     
     services:
     
     volumes:
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
HA_HISTORY_DB_ROOT_USER=admin
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
# We run in host mode to be able to be connected from HA.
    ports:
      - "8086:8086"
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
   Creating network "srv_default" with the default driver
   Creating volume "srv_ha-history-db-data" with default driver
   Creating volume "srv_ha-history-db-config" with default driver
   Creating ha-history-db ... done
   ```
   - Verify that the container is running with `sudo docker ps`. The output should look like the following:
   ```shell
   CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS                    NAMES
   5a8f45730d6d   influxdb:latest          "/entrypoint.sh infl…"   33 seconds ago   Up 31 seconds   0.0.0.0:8086->8086/tcp   ha-history-db
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
# We run in host mode to be able to be connected from HA.
    ports:
      - "3000:3000"
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
   Creating ha-grafana ... done
   ```
   - Verify that the container is running with `sudo docker ps`. The output should look like the following:
   ```shell
   CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS                    NAMES
   5a8f45730d6d   influxdb:latest          "/entrypoint.sh infl…"   33 seconds ago   Up 31 seconds   0.0.0.0:8086->8086/tcp   ha-history-db
   304599875ff0   grafana/grafana:latest   "/run.sh"                33 seconds ago   Up 31 seconds   0.0.0.0:3000->3000/tcp   ha-grafana
   ```

# Home Assistant

## Setup Home Assistant.

1. Through a web-browser logon as administrator to the installed Home Assistant.
2. Click on the name of the logged in user at the lower left corner:
   - Enable `Advanced mode`.
4. Go to `Configuration` -> `Add-ons, Backups & Supervisor` -> Click on the `Add-on store` at the lower right corner, and install the following add-ons (always set start on boot, watchdog to restart and update automatically):
   - `File Editor`:
     - We want to be able to edit files in the web-browser.
   - `Terminal & SSH`:
     - We want to be able to logon with ssh (logon-user is `root`).
     - Configure the add-on:
       - Set `Option` and `password` to a password specific for ssh-login (yes, not preferred, one should use authorized key instead).
       - Set `Network` to 22.
     - Restart the add-on.
   - `MariaDB`:
     - Configure the add-on:
       - Set `Option` and `password` to a password specific for the database.
       - We do not set any port as we do not want the database to be exposed outside the host.
5. Complete the MariaDB installation with:
   - Through the `File Editor` add-on, edit the file `File Editor` and add (change the string 'password' below to the right password:
     ```recorder:
        db_url: mysql://homeassistant:password@core-mariadb/homeassistant?charset=utf8mb4
     ```
   - Goto `Configuration` -> `Settings` -> `Server Controls` and press `Check Configuration`.
     - The output should state 'Configuration valid'. If not, change the recorder config above.
     - On the same page press `Restart` under `Server management`.
     - Once restarted go to `History` and if data is there, the MariaDB add-on is correctly configured.






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
