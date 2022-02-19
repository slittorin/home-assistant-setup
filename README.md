# Home Assistant - Setup

Although it would be possible to install HA Supervised on one RPI with all features, it would not be standard and would require alot of tweaks, and I want to keep the setup as standard and future proof as possible (we tried for instance [Install Home Assistant Supervised on RPI](https://peyanski.com/how-to-install-home-assistant-supervised-official-way/) on a single server).

We could also utilize one server with HASS.io and addons for MariaDB, InfluxDB and Grafana, but that would most likely not be future proof as the load will increase as more data is gathered.

Therefore we have gone for a two-server setup according to below.

## Table of content

- [Generic information](https://github.com/slittorin/home-assistant-setup#generic-information)
- [Governing principles](https://github.com/slittorin/home-assistant-setup#governing-principles)
- [Conceptual design](https://github.com/slittorin/home-assistant-setup#conceptual-design)
- [Setup for Server 1](https://github.com/slittorin/home-assistant-setup#setup-for-server-1)
  - [Preparation](https://github.com/slittorin/home-assistant-setup#preparation)
  - [Installation for InfluxDB](https://github.com/slittorin/home-assistant-setup#installation-for-influxdb)
  - [Installation for Grafana](https://github.com/slittorin/home-assistant-setup#installation-for-grafana)
- [Setup for Home Assistant](https://github.com/slittorin/home-assistant-setup#setup-for-home-assistant)
  - [Setup MariaDB](https://github.com/slittorin/home-assistant-setup#setup-mariadb)
  - [General setup](https://github.com/slittorin/home-assistant-setup#general-setup)
  - [History DB setup](https://github.com/slittorin/home-assistant-setup#history-db-setup)
  - [HACS](https://github.com/slittorin/home-assistant-setup#hacs).

## Generic information

For all changes to Home Assistant configuration files, you usually need to restart:
-  Goto `Configuration` -> `Settings` -> `Server Controls` and press `Check Configuration`.
   - The output should state 'Configuration valid'. If not, change the recorder config above.
   - On the same page press `Restart` under `Server management`.
- Any warnings or errors can be found in the file `/config/home-assistant.log`.
- The data is stored in the following tables:
  - `statistics` and `statistics_short_term`:
    - Added to HA in [2021.8.0](https://www.home-assistant.io/blog/2021/08/04/release-20218/#long-term-statistics).
    - `statistics` (data written for hourly data) is not purged presently purged. But we can expect this in the future?
    - `statistics_short_term` (data written for 5 minute data) is purged according to Recorder setting according to [statement](https://github.com/home-assistant/core/issues/64383#issuecomment-1015886985).

## Governing principles

EDIT 2022-01-16: Changed to ue only one InfluxDB bucket with no retention, instead of standard bucket for 53 weeks and down-sample data through InfluxDB capabilities into hourly measures in a separate bucket. This will be a configuration that will not require complexity with down-sampling.

#### Generic

- Keep to standard setup/config as much as possible.
- Limit the number of exposed ports/services on the Home Assistant.

#### Historical data and visualization

As of [2021.8.0](https://www.home-assistant.io/blog/2021/08/04/release-20218/#long-term-statistics) Home Assistant is be on the way to store and manage historical data (see information about tables above) in a better way, but is still in development phase, including capability to graphically visualize historical data.

However, as these capabilities are still in the early stages, specifically since not all integrations/capabilities use the historical data tables yet, and most importantly since the visualization capabilities in HA is not yet to a standard, we want to utilize historical data also through InfluxDB and Grafana for visualizations.

This entitles that we need to set governing principles to support the capabilities we required, for both sensors and where to store data.

##### We therefore define the following for storage and use of databases
- States and events in HA database (MariaDB):
  - We keep the data for 30 days (purge period).
  - We have a rather good setup that should cope with the load and volume of data, with the current number of sensors/integrations.
- Statistics tables in HA database (MariaDB) according to above.
  - If we want to utilize statistics, we need to ensure that the sensors will be able to be added to [Long term statistics](https://data.home-assistant.io/docs/statistics/).
    - Note that not all integrations and addons can utilize long term statistics yet, therefore we may need to create additional sensors to achieve this.
    - Ensure that sensors utilize `device_class`, `state_class` and `unit_of_measurement` according to [Configuration and governing principles](https://github.com/slittorin/home-assistant-configuration#governing-principles).
    - You can verify that data is written to the `statistics` table by verifying stored entities in the `statistics_meta` table according to [Exclude sensors for InfluxDB integration](https://github.com/slittorin/home-assistant-maintenance/blob/main/README.md#exclude-sensors-for-influxdb-integration).
- States also written to InfluxDB for historical data.
  - ~~Retention set to 53 weeks. No need to keep detailed state data for that long.~~
  - No retention for this bucket.
    - For sensors that write large amount of data, we may want to exlude this to be written to InfluxDB.
      - See [Exclude sensors for InfluxDB integration](https://github.com/slittorin/home-assistant-maintenance/blob/main/README.md#exclude-sensors-for-influxdb-integration).
      - If we want to have this type of data as historical data, this means that we need to create sensors that down-sample the sensors already in Home Assistant.\
  -~~Down-sample data through InfluxDB capabilities into hourly measures in a separate bucket.~~
    -~~No retention set for this bucket.~~

##### We therefore define the following for visualization
- For simplified visualization of sensor-states and historical data:
  - Utilize the built in HA functions and graphs.
  - Possibly with added graph-capability through Apex Charts.
- For more advanced visualization and historical data:
  - Utilize Grafana to retrieve data from:
    - InfluxDB:
      - For detailed data, from bucket that stores data in 53 weeks.
      - For hourly data, from bucket that contains down-sampled data (stored indefinitely).
    - HA database (MariaDB) and `statistics` table:
      - For hourly data (sum, min, max, medium).
    - We therefore have the possibility and capability to utilize two database sources for Grafana.

In the future, dependent on where HA platform will go, we may change the governing principles for storage and visualization of data.

#### Backup

- [GitHub Repository home-assistant-config](https://github.com/slittorin/home-assistant-config) is utilized for configuration files (push, on demand).
- Backup Home Assistant with snapshots (includes MariaDB database) according:
  - Daily snapshots, keep for 7 days (monday through saturday).
  - Weekly snapshots (sunday), keep for 8 weeks.
- We backup history database (InfluxDB) according:
  - Daily snapshots, keep for 7 days (monday through saturday).
  - Weekly snapshots (sunday), keep for 8 weeks.
- Files from the above are moved to my NAS for storage (old files deleted).

## Conceptual design

- homeassistant on VLAN-Server (192.168.3.20):
  - RPI 4 with 250GB SSD disk. Standard HASS.io install, with the following addons:
    - Setup a RHASS.io with [HASS.io install](https://github.com/slittorin/hass-io-install).
- server1 on VLAN-Server (192.168.3.30):
   - RPI 4 with 500GB SSD disk. Standard RPI, with docker and the following services:
     - InfluxDB.
     - Grafana.
   - Intended also to be utilized for other projects.
   - Setup a RPI instance with [Raspberry PI install](https://github.com/slittorin/raspberrypi-install/).

# Setup for Server 1

## Preparation

1. Install sysstat to be able to get system statistics with:
  ```bash
  sudo apt install sysstat
  ```
2. Under `/srv`:
   - Create the file `.env`.
   - Create the file `/srv/docker-compose.yml` with the following content:
     ```yaml
     version: '3'
     
     services:
     
     volumes:
     ```
3. We create a script that creates the necessary OS/HW statistics that we can store in files, that can be read by Home Assistant.
   - Create the following file `/srv/os-stats.sh`:
```bash
#!/bin/bash

# Purpose:
# This script saves OS/HW statistics in files to be read by Home Assistant.
# Script takes 15 minuts to run.
# Put in cron to be run every 15 minutes.
#
# Requires sysstat to be installed.
#
# Statistics is saved to:
# /srv/stats/disk_used_pct.txt          - Disk utilization in percent.
# /srv/stats/mem_used_pct.txt           - RAM utilization in percent.
# /srv/stats/swap_used_pct.txt          - Swap utilization in percent.
# /srv/stats/cpu_used_pct.txt           - CPU utilization in percentage over 14 minutes and 55 seconds.
#
# Usage:
# ./os-stats.sh

# Load environment variables (mainly secrets).
if [ -f "/srv/.env" ]; then
    export $(cat "/srv/.env" | grep -v '#' | sed 's/\r$//' | awk '/=/ {print $1}' )
fi

# Variables:
base_dir="/srv"
stats_dir="/srv/stats"

_initialize() {
    cd "${base_dir}"
    mkdir -p ${stats_dir}
}

# Pct used for specific mount directry, usually /
_disk_used() {
   MOUNT_DIR="/"
   USED_PCT=`df -m ${MOUNT_DIR} | tail -1 | awk '{ print $5 }' | sed 's/%//'`
   echo "${USED_PCT}" > ${stats_dir}/disk_used_pct.txt
}

# Ram used by the system.
_ram_used() {
   USED_PCT=`free -m | grep "Mem:" | awk '{ printf("%.1f", (($2-$4) / $2)*100) }'`
   echo "${USED_PCT}" > ${stats_dir}/mem_used_pct.txt
}

# Swap used by the system.
_swap_used() {
   USED_PCT=`free -m | grep "Swap:" | awk '{ printf("%.1f", (($2-$4) / $2)*100) }'`
   echo "${USED_PCT}" > ${stats_dir}/swap_used_pct.txt
}

# CPU percentage retrieved every 5 seconds for 180 times.
# This gives the load average over 15 minutes.
_cpu_used() {
   USED_PCT=`sar 5 180 | grep "Average" | awk '{ printf("%.f", (100-$8)) }'`
   echo "${USED_PCT}" > ${stats_dir}/cpu_used_pct.txt
}

_finalize() {
    exit 0
}

# Main
_initialize
_disk_used
_ram_used
_swap_used
_cpu_used
_finalize
```
   - Make it executable with `chmod ug+x os-stats.sh`.
   - Add it to crontab with `sudo crontab -e` to run every 15 minutes by adding: `*/15 * * * * /srv/os-stats.sh`

## Installation for InfluxDB

1. Check versions of available docker images for InfluxDB at [Docker - InfluxDB](https://hub.docker.com/_/influxdb).
   - If you do not want the 'latest' version, use version number.
   - At time of writing (20220207) the 'latest' version is 2.1.1 (isolated with `sudo docker image inspect influxdb` and looking for 'INFLUXDB_VERSION').
2. Create the directory `/srv/ha-history-db`, and the following sub-directories:
   - `backup`.
3. For the file `/srv/.env` add the following content:
   - `HA_HISTORY_DB_ROOT_PASSWORD` - Chose a complex and long password.
   - `HA_HISTORY_DB_ROOT_TOKEN` - Can only be added after the InfluxDB instance has been setup (Data -> API Tokens -> admin's Token), see below.
   - `HA_HISTORY_DB_GRAFANA_TOKEN` - Can only be added after the InfluxDB instance has been setup (Data -> API Tokens -> admin's Token), see below.
```
HA_HISTORY_DB_HOSTNAME=localhost
HA_HISTORY_DB_ROOT_USER=admin
HA_HISTORY_DB_ROOT_PASSWORD=[not shown here]
HA_HISTORY_DB_ROOT_TOKEN=[not shown here]
HA_HISTORY_DB_GRAFANA_TOKEN=not shown here]
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
      - "/srv/ha-history-db/backup:/backup"
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
7. In a web browser go the IP address (or hostname) of server1 and port 8086, for example [http://192.168.2.30:8086/](http://192.168.2.30:8086/).
   - Through 'Data -> API Tokens -> admin's Token', copy the token and add to `HA_HISTORY_DB_ROOT_TOKEN` in `/srv/.env`.
   -  Go to `Data` -> `API Tokens` -> `Generate API Token` -> `Read/Write API Token`:
      - Set a descriptive name: `Read to HA bucket`
      - Choose bucket `ha` for both read, but remove from write.
      - Choose the newly created token and copy the token and add to `HA_HISTORY_DB_GRAFANA_TOKEN` in `/srv/.env`.
8. Create the following backup-script `/srv/backup-influxdb.sh` to take InfluxDB-backup through docker-compose (remember to set `chmod ugo+x`).
```bash
#!/bin/bash

# Inspired by: https://gist.github.com/mihow/9c7f559807069a03e302605691f85572
#
# Purpose:
# This script backs up full influx according to:
# - Daily snapshots, keep for 7 days (monday through saturday).
# - Weekly snapshots (sunday), keep for 8 weeks.
#
# Usage:
# ./influxdb-backup.sh

# Load environment variables (mainly secrets).
if [ -f "/srv/.env" ]; then
    export $(cat "/srv/.env" | grep -v '#' | sed 's/\r$//' | awk '/=/ {print $1}' )
fi

# Variables:
container="ha-history-db"
base_dir="/srv"
docker_compose_file="${base_dir}/docker-compose.yml"
influxdb_logfile="${base_dir}/influxdb-backup.log"
influxdb_backup_dir="${base_dir}/${container}/backup/backup.tmp"
influxdb_backup_container_dir="/backup/backup.tmp"
influxdb_backup_dest="${base_dir}/${container}/backup/"

# Set name and retention according day of week.
# Default is daily backup.
day_of_week=$(date +%u)
influxdb_backup_pre="influxdb-backup-daily"
retention_days=7
if [[ "$day_of_week" == 7 ]]; then # On sundays.
    influxdb_backup_pre="influxdb-backup-weekly"
    retention_days=56 # 8 weeks.
fi
influxdb_backup_filename="${influxdb_backup_pre}-$(date +%Y%m%d_%H%M%S)"

_initialize() {
    cd "${base_dir}"
    touch "${influxdb_logfile}"

    echo ""
    echo "$(date +%Y%m%d_%H%M%S): Starting InfluxDB Backup."

    rm -r "${influxdb_backup_dir}/"
    mkdir "${influxdb_backup_dir}"
}

_influxdb_backup() {
    echo "$(date +%Y%m%d_%H%M%S): Backing up..."
    docker-compose -f "${docker_compose_file}" exec -T "${container}" influx backup "${influxdb_backup_container_dir}" -t "${HA_HISTORY_DB_ROOT_TOKEN}"

    echo "$(date +%Y%m%d_%H%M%S): Compressing Backup..."
    tar_file="${influxdb_backup_dest}${influxdb_backup_filename}.tar"
    tar -cvf "${tar_file}" "${influxdb_backup_dir}/"
    echo "$(date +%Y%m%d_%H%M%S): Compressed backup to: ${tar_file}"

    echo "$(date +%Y%m%d_%H%M%S): Backup done."
}

_influxdb_cleanup() {
    find "${influxdb_backup_dest}" -name "${influxdb_backup_pre}-*" -mtime +${retention_days} -delete
    echo "$(date +%Y%m%d_%H%M%S): Done retention cleanup to ${retention_days} days for filenames starting with ${influxdb_backup_pre}-"
}

_finalize() {
    echo "$(date +%Y%m%d_%H%M%S): Finished InfluxDB Backup."
    exit 0
}

# Main
_initialize >> "${influxdb_logfile}" 2>&1
_influxdb_backup >> "${influxdb_logfile}" 2>&1
_influxdb_cleanup >> "${influxdb_logfile}" 2>&1
_finalize >> "${influxdb_logfile}" 2>&1
```
9. Create the following crontab entry with `sudo crontab -e` to run the script each day at 00:00:01: `1 0 * * * /srv/backup-influxdb.sh`.
10. Verify that the crontab is correct with `crontab -l` (run in the context of user 'pi').
11. Wait to the day after and check the log-file `/srv/backup-influxdb.log` /and backup-directory `/srv/ha-history-db/backup` so that backups are taken.

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
    restart: always
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
7. In web browser go the IP address (or hostname) of server1 and port 3000, for example [http://192.168.2.30:3000/](http://192.168.2.30:3000/) (yes, we setup grafana to utilize no credentials).
   - Go to `Configuration` -> `Data sources` -> `Add new data source`:
     - Choose `InfluxDB`:
     - Set `Name`: `ha_history_db`.
     - Choose `Query language`: `Flux`.
     - Set `URL`: `http://192.168.2.30:8086`.
     - Check `Skip TLS Verify`.
     - Under `InfluxDB Details`:
       - Set `Organization`: `lite`
       - Set `Token` to read-token created in step 7 for the setup of InfluxDB.
       - Set `Default bucket`: `ha`
       - Press `Save and test`.
         - The result should be `1 buckets found`.

# Setup for Home Assistant.

For all changes to Home Assistant configuration files, you usually need to restart:
-  Goto `Configuration` -> `Settings` -> `Server Controls` and press `Check Configuration`.
   - The output should state 'Configuration valid'. If not, change the recorder config above.
   - On the same page press `Restart` under `Server management`.
- Any warnings or errors can be found in the file `/config/home-assistant.log`.

## Setup MariaDB

1. Go to `Configuration` -> `Add-ons, Backups & Supervisor` -> Click on the `Add-on store` at the lower right corner, and install the following add-ons (always set start on boot, watchdog to restart and update automatically):
   - `MariaDB`:
     - Configure the add-on:
       - Set `Option` and `password` to a password specific for the database.
       - We do not set any port as we do not want the database to be exposed outside the host.
2. Through the `File Editor` add-on, edit the file `/config/secrets.yaml` and add (change the string 'password' below to the right password):
   `recorder_db_url: mysql://homeassistant:password@core-mariadb/homeassistant?charset=utf8mb4`
3. Through the `File Editor` add-on, edit the file [/config/configuration.yaml](https://github.com/slittorin/home-assistant-config/blob/master/configuration.yaml) and add:
   ```yaml
   recorder:
     db_url: !secret recorder_db_url
   ```
4. Verify the config-file and restart.
5. Verify that the setup is working correct by looking in the Home Assistant logfile.

## General setup

1. Through a web-browser logon as administrator to the installed Home Assistant.
2. Click on the name of the logged in user at the lower left corner:
   - Enable `Advanced mode`.
3. Go to `Configuration` -> `Add-ons, Backups & Supervisor` -> Click on the `Add-on store` at the lower right corner, and install the following add-ons (always set start on boot, watchdog to restart and update automatically):
   - `File Editor`:
     - We want to be able to edit files in the web-browser.
   - `Terminal & SSH`:
     - We want to be able to logon with ssh (logon-user is `root`).
     - Configure the add-on:
       - Set `Option` and `password` to a password specific for ssh-login (yes, not preferred, one should use authorized key instead).
       - Set `Network` to 22.
     - Restart the add-on.
   - `Samba share`:
     - Click `Install` and then `Start`, where error will occur, press `Failed to start add-on - Configuration validation failed`:
       - Set `password` in the config file.
       - Leave the rest as it is.
     - Press `Start`.
4. We enable to add resources to lovelace:
   - With `File Editor` create the following directories:
     - `/config/www`.
5. For readability, as will have lots of configuration data, we create separate yaml-files:
   - With `File Editor` create the following directories:
     - `/config/logs`.
     - `/config/scripts`.
     - `/config/packages`.
     - `/config/custom_components`.
   - With `File Editor` add the following files in directory `/config`:
     - `.env`
     - `sensors.yaml`
6. Through the `File Editor` add-on, edit the file [/config/configuration.yaml](https://github.com/slittorin/home-assistant-config/blob/master/configuration.yaml) and add at the bottom of the file:
     - `allowlist_external_dirs` is required to get sensor.file to get last line of log-files.
     ```yaml
     homeassistant:
       allowlist_external_dirs:
         - '/config/logs'
       
       packages:
     ```
7. Through the `File Editor` add-on, edit the file [/config/configuration.yaml](https://github.com/slittorin/home-assistant-config/blob/master/configuration.yaml) and add before the rows with `!include`:
     ```yaml
     sensor: !include sensors.yaml
     ```
8. We setup logging to log warning and above.
   - Through the `File Editor` add-on, edit the file [/config/configuration.yaml](https://github.com/slittorin/home-assistant-config/blob/master/configuration.yaml) and add:
     ```yaml
     logger:
       default: warning
     ```
9. Setup Recorder correctly to keep data in database for 30 days, and write every 10:th second to the database to reduce load (even though we do not need it since we have an SSD disk instead of SD Card), and ensure that logbook is enabled:
   - Through the `File Editor` add-on, edit the file [/config/configuration.yaml](https://github.com/slittorin/home-assistant-config/blob/master/configuration.yaml) and add after `recorder:`:
     ```yaml
         purge_keep_days: 30
         auto_purge: true
         commit_interval: 10
     
     logbook:
     ```
10. Add areas that is representative for your home.
   - Go to `Configuration` -> `Devices and services` -> `Areas` and update the rooms/areas that represent your home.
11. Verify the config-file and restart.
12. Verify that the setup is working correct by looking in the Home Assistant logfile.

## History DB setup

1. In a web browser go the IP address (or hostname) of server1 and port 8086, for example [http://192.168.2.30:8086/](http://192.168.2.30:8086/).
   - Login with the username and password setup above.
   - Go to `Data` -> `API Tokens` -> `Generate API Token` -> `Read/Write API Token`:
     - Set a descriptive name: `Read/Write to HA bucket`
     - Choose bucket `ha` for both read and write.
   - Choose the newly created token and copy the token.
2. Through the `File Editor` add-on, edit the file `/config/secrets.yaml` and add (change the string 'token' below to the right token):
   `history_db_token: TOKEN`.
3. Through the `File Editor` add-on, edit the file [/config/configuration.yaml](https://github.com/slittorin/home-assistant-config/blob/master/configuration.yaml) and add:
```yaml

history:
  exclude:
    domains:
      # Updater: Is not required in history data.
      - updater
    entity_globs:
      # We do not include any of the test_-sensors.
      - binary_sensor.test_*
      - sensor.test_*

influxdb:
  api_version: 2
  ssl: false
  host: 192.168.2.30
  port: 8086
  organization: lite
  bucket: ha
  token: !secret history_db_token
  max_retries: 3
```
4. Verify that data is written to the InfluxDB_bucket with:
   - In web browser go the IP address (or hostname) of server1 and port 8086, for example [http://192.168.2.30:8086/](http://192.168.2.30:8086/).
     - Check that data is written to the ha-bucket.
     - If not, find the error and correct.

## HACS

We want to be able to download more from Home Assistant Community Store (HACS).\
HACS is an integration that needs to be installed according to [HACS site](https://hacs.xyz/).

Follow the instructions for Supervisor install:
1. Open `Terminal` and go to directory `/config`:
   - Run the following command: `wget -O - https://get.hacs.xyz | bash -`.
   - The installation should state that installation is complete, and that HA can be restarted.
2. Restart the Home Assistant server under `Server management`.
3. Clear your browser cache, yep, important!.
4. Reconnect to HA.
5. Add the integration 'HACS':
   - Follow the instructions for [Configuration of HACS](https://hacs.xyz/docs/configuration/basic).
     - In my case, I utilized my already existing github account to retrieve the token.
6. On Integration page, click on `Configure` on the HACS integration:
   - Enabled AppDaemon and NetDaemon apps.
7. Restart the Home Assistant server under `Server management`.
