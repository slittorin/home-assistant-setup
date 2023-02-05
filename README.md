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
  - [Git for shell-scripts]()
  - [OS/HW Statistics](https://github.com/slittorin/home-assistant-setup#oshw-statistics)
  - [Docker volume sizes](https://github.com/slittorin/home-assistant-setup/blob/main/README.md#docker-volume-sizes).
  - [Installation for InfluxDB](https://github.com/slittorin/home-assistant-setup#installation-for-influxdb)
    - [Backup for InfluxDB](https://github.com/slittorin/home-assistant-setup#backup-for-influxdb)
  - [Installation for Grafana](https://github.com/slittorin/home-assistant-setup#installation-for-grafana)
    - [Backup for Grafana](https://github.com/slittorin/home-assistant-setup#backup-for-grafana-database)
    - [Git for Grafana](https://github.com/slittorin/home-assistant-setup#git-for-grafana)
  - [Backup of Server1 to NAS](https://github.com/slittorin/home-assistant-setup/blob/main/README.md#backup-of-server1-to-nas)
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

- We push, on demand, all HA config files to [GitHub Repository home-assistant-config](https://github.com/slittorin/home-assistant-config).
  - See [Home Assistant Configuration Github push](https://github.com/slittorin/home-assistant-configuration#github-push).
- Backup Home Assistant with snapshots (includes MariaDB database) according:
  - Daily snapshots, keep for 7 days (monday through saturday).
  - Weekly snapshots (sunday), keep for 8 weeks (57 days).
  - See [Home Assistant Configuration Backup](https://github.com/slittorin/home-assistant-configuration#backup)
  - Copy backup files daily to server 1 each morning: 
    - See [Home Assistant Configuration Copy backup files to server 1](https://github.com/slittorin/home-assistant-configuration#copy-backup-files-to-server1).
    - Can also be triggered manually.
- We backup Grafana database according:
  - Daily snapshots, keep for 7 days (monday through saturday).
  - Weekly snapshots (sunday), keep for 8 weeks.
  - See [Backup for Grafana](https://github.com/slittorin/home-assistant-setup#backup-for-grafana-database) below.
- Git for Grafana dashboards.
  - Triggered from Home Assistant.
  - See [Git for Grafana](https://github.com/slittorin/home-assistant-setup#git-for-grafana) below.
- We backup history database (InfluxDB) according:
  - Daily snapshots, keep for 7 days (monday through saturday).
  - Weekly snapshots (sunday), keep for 8 weeks.
  - See [Backup for InfluxDB](https://github.com/slittorin/home-assistant-setup#backup-for-influxdb) below.
- We backup our RPI-server that managed InfluxDB and Grafana.
  - Since the [SSD-failure](https://github.com/slittorin/home-assistant-maintenance/blob/main/README.md#failed-ssd-drive) we need to backup files from Server1.
  - We do this by copying the files with rsync a specific share on my NAS.

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
- (existing) NAS on VLAN-Server (192.168.3.10).
  - Specific share `server-backup`, without quota, recycle bin or throttle limits.
  - Specific user `pi-backup` that has RW-access to the above share.

# Setup for Server 1

## Preparation

1. Install sysstat to be able to get system statistics and fiddle with json with:
  ```bash
  sudo apt install sysstat
  sudo apt install jq
  ```
2. Under `/srv`:
   - Create the file `.env`.
   - Create the file `/srv/docker-compose.yml` with the following content:
     ```yaml
     version: '3'
     
     services:
     
     volumes:
     ```
3. Under `/srv`:
   - Create the directory `log`.
   - Create the directory `ha/backup`.
   - Set right permissions to `backup` with `sudo chmod go+rw backup`.
4. To all HA to utilize shell on server1, setup key-based authentication for SSH:
   - Note that this step can only be performed after home assistant server is [setup with ssh access](https://github.com/slittorin/home-assistant-setup#general-setup):
   - On home assistant server with ssh:
     - At the home directory for user root, generate at key with `ssh-keygen -t rsa`:
       - Press enter for file to save the key (/root/.ssh/id_rsa).
       - Press enter for passphrase (empty passphrase).
     - Transfer Your Public Key to the home assistant server with `ssh-copy-id pi@192.168.2.30`:
       - Enter `yes` at 'Are you sure you want to continue connecting'.
       - Enter the password for ssh, at server1.
     - The public key is now copied, verify with `ssh pi@192.168.2.30` that should obtain shell without password.
     - Ensure that the permissions are correct in `/root/.ssh` with `chmod 700 *`.
     - Copy all ssh files to `/config/.ssh` from `/root/.ssh` to allow these to be preserved (may otherwise be removed during upgrades):
       - `mkdir /config/.ssh`
       - `cd /config/.ssh`
       - `cp /root/.ssh/* .`

## Git for shell-scripts

We want to utilize Github Push to push shell-scripts from server1.

Pre-requisities: That repository `grafana-config` is created in Github.

Perform the following:

1. If not already done, create a Github personal access token:
   - According [instructions](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token).
   - Set preferably an Expiration date (and keep note of the date and update the token accordingly).
   - Set the following scopes:
     - `repo`.
     - `gists`.
   - Copy the token.
2. Through the `File Editor` add-on, add the following to `.env` in `/srv/` on HA-server (where TOKEN is the copied token, ensure that username and repository is correct):
   `GITHUB_CONNECT_STRING_SERVER1=https://TOKEN@github.com/slittorin/server1-config`
3. Through the `File Editor` add-on, create `.gitignore` in `/srv/` with the following content:
```git config
# .gitignore for server1.

# First rule: Ignore everything, all files in root and all directories.
*
*/*

# Third rule: Whitelisted files, these will not be ignored.
!*.yml
!.gitignore
!*.sh

# Last rule: If we make a mistake above, ensure these files are ignored, otherwise your secret data/credentials will leak.
.env
```
4. Run the following in the `/srv/` directory (change your email and name to match your Github account, and ensure that all directories you want to add to Github are present):\
   You will be asked to provide a comment at commit.
```bash
sudo git init
sudo git config user.email "you@example.com"
sudo git config user.name "Your Name"
```
5. Run the following in the `/srv/` directory (where TOKEN is the copied token, ensure that usename and repository is correct):\
   Note that git push can give error, that is nothing to trouble-shoot.
```bash
sudo git remote add origin https://TOKEN@github.com/slittorin/server1-config
sudo git push -u origin master
```

## OS/HW statistics

We want to track OS/HW statistics that can be pulled into HA.

1. We create a script that creates the necessary OS/HW statistics that we can store in files, that can be read by Home Assistant.
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
# /srv/stats/cpu_used_pct.txt           - CPU utilization in percentage over 15 minutes.
# /srv/stats/cpu_temp.txt               - CPU temperature in degrees celcius.
# /srv/stats/uptime.txt                 - Uptime since (last reboot).
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
   echo "$(date +%Y%m%d_%H%M%S),${USED_PCT}" > ${stats_dir}/disk_used_pct.txt
}

# Ram used by the system.
_ram_used() {
   USED_PCT=`free -m | grep "Mem:" | awk '{ printf("%.1f", (($2-$4) / $2)*100) }'`
   echo "$(date +%Y%m%d_%H%M%S),${USED_PCT}" > ${stats_dir}/mem_used_pct.txt
}

# Swap used by the system.
_swap_used() {
   USED_PCT=`free -m | grep "Swap:" | awk '{ printf("%.1f", (($2-$4) / $2)*100) }'`
   echo "$(date +%Y%m%d_%H%M%S),${USED_PCT}" > ${stats_dir}/swap_used_pct.txt
}

# CPU temp.
# Works for RPI.
_cpu_temp() {
   USED_PCT=`cat /sys/class/thermal/thermal_zone0/temp | awk '{ printf("%.f", $1/1000) }'`
   echo "$(date +%Y%m%d_%H%M%S),${USED_PCT}" > ${stats_dir}/cpu_temp.txt
}

# System up since.
_uptime() {
   UPTIME=`uptime -s`
   echo "$(date +%Y%m%d_%H%M%S),${UPTIME}" > ${stats_dir}/uptime.txt
}

# CPU percentage retrieved every 5 seconds for 180 times.
# This gives the load average over 15 minutes. I.e. script runs for 15 minutes.
_cpu_used() {
   USED_PCT=`sar 5 180 | grep "Average" | awk '{ printf("%.f", (100-$8)) }'`
   echo "$(date +%Y%m%d_%H%M%S),${USED_PCT}" > ${stats_dir}/cpu_used_pct.txt
}

_finalize() {
    exit 0
}

# Main
_initialize
_disk_used
_ram_used
_swap_used
_cpu_temp
_uptime
_cpu_used
_finalize
```
2. Make it executable with `chmod ug+x os-stats.sh`.
3. Add it to crontab with `sudo crontab -e` to run every 15 minutes by adding: `*/15 * * * * /srv/os-stats.sh`
4. Check with `sudo crontab -l` that the row was added.

## Docker volume sizes

We want to track docker volume size statistics, that can be pulled into HA.

1. We create a script that creates the necessary docker volume size statisticsls that we can store in files, that can be read by Home Assistant.
   - Create the following file `/srv/docker-volume-sizes.sh`:
```bash
#!/bin/bash

# Inspired by: https://medium.com/homullus/how-to-inspect-volumes-size-in-docker-de1068d57f6b
#

# Purpose:
# This script lists all docker containers volumes and sizes.
# Data is written to log-file and to comma separated file.
# Sizes are in MB.
#
# Usage:
# ./docker_volume_sizes.sh

# Load environment variables (mainly secrets).
if [ -f "/srv/.env" ]; then
    export $(cat "/srv/.env" | grep -v '#' | sed 's/\r$//' | awk '/=/ {print $1}' )
fi

# Variables:
base_dir="/srv"
stats_dir="/srv/stats"
statsfile="${stats_dir}/docker_volume_sizes.txt"
logfile="${base_dir}/log/docker-volume-sizes.log"
logfile_tmp="${base_dir}/log/docker-volume-sizes.tmp"
timestamp="$(date +%Y%m%d_%H%M%S)"

_initialize() {
    cd "${base_dir}"
    touch "${logfile}"
    > "${statsfile}"

    echo ""
    echo "$(date +%Y%m%d_%H%M%S): Starting Docker volume sizes."
}

_volume_sizes() {
    for DOCKER_ID in `docker ps -a | awk '{ print $1 }' | tail -n +2`; do
        DOCKER_NAME=`docker inspect -f {{.Name}} ${DOCKER_ID}`
        echo "$(date +%Y%m%d_%H%M%S): For docker container: ${DOCKER_NAME} (${DOCKER_ID})"

        VOLUME_IDS=$(docker inspect -f "{{.Config.Volumes}}" ${DOCKER_ID})
        VOLUME_IDS=$(echo ${VOLUME_IDS} | sed 's/map\[//' | sed 's/]//')

        ARRAY=(${VOLUME_IDS// / })
        for i in "${!ARRAY[@]}"; do
            VOLUME_ID=$(echo ${ARRAY[i]} | sed 's/:{}//')
            VOLUME_SIZE=`docker exec -i ${DOCKER_NAME} du -d 0 -m ${VOLUME_ID} | awk '{ print $1 }'`

            echo "$(date +%Y%m%d_%H%M%S): Size in MB for volume ${VOLUME_ID}: ${VOLUME_SIZE}"

            echo "${timestamp},${DOCKER_NAME},${VOLUME_ID},${VOLUME_SIZE}" >> "${statsfile}"
        done
    done
}

_finalize() {
    echo "$(date +%Y%m%d_%H%M%S): Finished Docker volume sizes."

    tail -n10000 ${logfile} > ${logfile_tmp}
    rm ${logfile}
    mv ${logfile_tmp} ${logfile}

    exit 0
}

# Main
_initialize >> "${logfile}" 2>&1
_volume_sizes >> "${logfile}" 2>&1
_finalize >> "${logfile}" 2>&1
```
2. Make it executable with `chmod ug+x docker-volume-sizes.sh`.
3. Add it to crontab with `sudo crontab -e` to run once each hour by adding: `0 * * * * /srv/docker-volume-sizes.sh`
4. Check with `sudo crontab -l` that the row was added.

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
      - INFLUXDB_LOGGING_FORMAT=auto
      - INFLUXDB_LOGGING_LEVEL=warn
      - INFLUXDB_LOGGING_SUPPRESS_LOGO=true
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
      - Choose bucket `ha` for read, but remove from write.
      - Choose the newly created token and copy the token and add to `HA_HISTORY_DB_GRAFANA_TOKEN` in `/srv/.env`.

### Backup for InfluxDB

1. Create the following backup-script `/srv/influxdb-backup.sh` to take InfluxDB-backup through docker-compose (remember to set `chmod ugo+x`).
   Updated 2023-01-16 with error-management.
```bash
#!/bin/bash

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
logfile="${base_dir}/log/influxdb-backup.log"
logfile_tmp="${base_dir}/log/influxdb-backup.tmp"
backup_dir="${base_dir}/${container}/backup/backup.tmp"
backup_container_dir="/backup/backup.tmp"
backup_dest="${base_dir}/${container}/backup/"
error_occured=0
error_message=""

# Set name and retention according day of week.
# Default is daily backup.
day_of_week=$(date +%u)
backup_pre="influxdb-backup-daily"
retention_days=7
if [[ "$day_of_week" == 7 ]]; then # On sundays.
    backup_pre="influxdb-backup-weekly"
    retention_days=57 # 8 weeks + 1 day.
fi
backup_filename="${backup_pre}-$(date +%Y%m%d_%H%M%S)"

_initialize() {
    cd "${base_dir}"
    touch "${logfile}"

    echo ""
    echo "$(date +%Y%m%d_%H%M%S): Starting InfluxDB backup."

    rm -r "${backup_dir}/"
    mkdir "${backup_dir}"
}

_backup() {
    echo "$(date +%Y%m%d_%H%M%S): Backup of influxdb started."
    RESULT=`docker-compose -f "${docker_compose_file}" exec -T "${container}" influx backup "${backup_container_dir}" -t "${HA_HISTORY_DB_ROOT_TOKEN}"`
    RESULT_CODE=$?
    if [ ${RESULT_CODE} -ne 0 ]; then
       error_occured=1
       error_message="influx backup error"
       echo "$(date +%Y%m%d_%H%M%S): ERROR. ${error_message}. Exit code: ${RESULT_CODE}: ${RESULT}"
    else
       echo "$(date +%Y%m%d_%H%M%S): Backup of influxdb performed."
    fi
}

_compress() {
    if [ ${error_occured} -eq 0 ]; then
           echo "$(date +%Y%m%d_%H%M%S): Compress of backup started."
           tar_file="${backup_dest}${backup_filename}.tar"
           RESULT=`tar -cvf "${tar_file}" "${backup_dir}/"`
           RESULT_CODE=$?
           if [ ${RESULT_CODE} -ne 0 ]; then
              error_occured=1
              error_message="tar command error when compressing"
              echo "$(date +%Y%m%d_%H%M%S): ERROR. ${error_message}. Exit code: ${RESULT_CODE}: ${RESULT}"
           else
         echo "$(date +%Y%m%d_%H%M%S): Compress of backup performed."
           fi
        fi
}

_cleanup() {
    if [ ${error_occured} -eq 0 ]; then
           echo "$(date +%Y%m%d_%H%M%S): Retention of files started."
           RESULT=`find "${backup_dest}" -name "${backup_pre}-*" -mtime +${retention_days} -delete`
           RESULT_CODE=$?
           if [ ${RESULT_CODE} -ne 0 ]; then
              error_occured=1
              error_message="Error when removing files (retention)"
              echo "$(date +%Y%m%d_%H%M%S): ERROR. ${error_message}. Exit code: ${RESULT_CODE}: ${RESULT}"
           else
         echo "$(date +%Y%m%d_%H%M%S): Retention of files performed to ${retention_days} days for filenames starting with ${backup_pre}-"
           fi
        fi
}

_finalize() {
    if [ ${error_occured} -eq 0 ]; then
       echo "$(date +%Y%m%d_%H%M%S): Finished InfluxDB backup. No error."

       tail -n10000 ${logfile} > ${logfile_tmp}
       rm ${logfile}
       mv ${logfile_tmp} ${logfile}

       exit 0
    else
       echo "$(date +%Y%m%d_%H%M%S): Exited InfluxDB backup. ERROR: ${error_message}."

       tail -n10000 ${logfile} > ${logfile_tmp}
       rm ${logfile}
       mv ${logfile_tmp} ${logfile}

       exit 1
    fi
}

# Main
_initialize >> "${logfile}" 2>&1
_backup >> "${logfile}" 2>&1
_compress >> "${logfile}" 2>&1
_cleanup >> "${logfile}" 2>&1
_finalize >> "${logfile}" 2>&1
```
2. Create the following crontab entry with `sudo crontab -e` to run the script each day at 00:02:00: `* 1 * * * /srv/influxdb-backup.sh`.
3. Verify that the crontab is correct with `sudo crontab -l` (run in the context of user 'pi').
4. Wait to the day after and check the log-file `/srv/influxdb-backup.log` and backup-directory `/srv/ha-history-db/backup` so that backups are taken.

## Installation for Grafana

1. Check versions of available docker images for Grafana at [Docker - Grafana](https://hub.docker.com/r/grafana/grafana).
   - If you do not want the 'latest' version, use version number, or use 'main'.
   - At time of writing (20220207) the 'latest' version is 8.3.3 (isolated with running the command `/usr/share/grafana/bin/grafana-server -v` on the container).
2. Create the directory `/srv/ha-grafana`, and the following sub-directories:
   - `json`.
   - `backup`.
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
      - "/srv/ha-grafana/backup:/backup"
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

### Backup for Grafana Database

1. Create the following backup-script `/srv/grafana-backup.sh` to take Grafana-backup of the Sqlite-database file (remember to set `chmod ugo+x`).
   Updated 2023-01-16 with error-management.
```bash
#!/bin/bash

# This script backs up full Grafana according to:
# - Daily snapshots, keep for 7 days (monday through saturday).
# - Weekly snapshots (sunday), keep for 8 weeks.
#
# Usage:
# ./grafana-backup.sh

# Load environment variables (mainly secrets).
if [ -f "/srv/.env" ]; then
    export $(cat "/srv/.env" | grep -v '#' | sed 's/\r$//' | awk '/=/ {print $1}' )
fi

# Variables:
# -----------------------------------------------------------------
container="ha-grafana"
base_dir="/srv"
docker_compose_file="${base_dir}/docker-compose.yml"
logfile="${base_dir}/log/grafana-backup.log"
logfile_tmp="${base_dir}/log/grafana-backup.tmp"
backup_dir="${base_dir}/${container}/backup/backup.tmp"
backup_container_dir="/backup/backup.tmp"
backup_dest="${base_dir}/${container}/backup/"
error_occured=0
error_message=""

# Set name and retention according day of week.
# Default is daily backup.
# -----------------------------------------------------------------
day_of_week=$(date +%u)
backup_pre="grafana-backup-daily"
retention_days=7
if [[ "$day_of_week" == 7 ]]; then # On sundays.
    backup_pre="grafana-backup-weekly"
    retention_days=57 # 8 weeks + 1 day.
fi
backup_filename="${backup_pre}-$(date +%Y%m%d_%H%M%S)"

_initialize() {
    cd "${base_dir}"
    touch "${logfile}"

    echo ""
    echo "$(date +%Y%m%d_%H%M%S): Starting Grafana Backup."

    rm -r "${backup_dir}/"
    mkdir "${backup_dir}"
}

_backup() {
    echo "$(date +%Y%m%d_%H%M%S): Copy of grafana.db started."
    RESULT=`docker cp "${container}:/var/lib/grafana/grafana.db" "${backup_dir}"`
    RESULT_CODE=$?
    if [ ${RESULT_CODE} -ne 0 ]; then
       error_occured=1
       error_message="docker cp error"
       echo "$(date +%Y%m%d_%H%M%S): ERROR. ${error_message}. Exit code: ${RESULT_CODE}: ${RESULT}"
    else
       echo "$(date +%Y%m%d_%H%M%S): Copy of grafana.db performed."
    fi

    if [ ${error_occured} -eq 0 ]; then
       echo "$(date +%Y%m%d_%H%M%S): Compression of backup started."
       tar_file="${backup_dest}${backup_filename}.tar"
       RESULT=`tar -cvf "${tar_file}" "${backup_dir}/"`
       RESULT_CODE=$?
       if [ ${RESULT_CODE} -ne 0 ]; then
          error_occured=1
          error_message="tar command error when compressing"
          echo "$(date +%Y%m%d_%H%M%S): ERROR. ${error_message}. Exit code: ${RESULT_CODE}: ${RESULT}"
       else
          echo "$(date +%Y%m%d_%H%M%S): Compression of backup performed to: ${tar_file}"
       fi
   fi
}

_cleanup() {
    if [ ${error_occured} -eq 0 ]; then
       echo "$(date +%Y%m%d_%H%M%S): Retention of files started."
       RESULT=`find "${backup_dest}" -name "${backup_pre}-*" -mtime +${retention_days} -delete`
       RESULT_CODE=$?
       if [ ${RESULT_CODE} -ne 0 ]; then
          error_occured=1
          error_message="Error when removing files (retention)"
          echo "$(date +%Y%m%d_%H%M%S): ERROR. ${error_message}. Exit code: ${RESULT_CODE}: ${RESULT}"
       else
          echo "$(date +%Y%m%d_%H%M%S): Retention of files performed to ${retention_days} days, for filenames starting with ${backup_pre}-"
       fi
    fi
}

_finalize() {
    if [ ${error_occured} -eq 0 ]; then
       echo "$(date +%Y%m%d_%H%M%S): Finished Grafana backup. No error."

       tail -n10000 ${logfile} > ${logfile_tmp}
       rm ${logfile}
       mv ${logfile_tmp} ${logfile}

       exit 0
    else
       echo "$(date +%Y%m%d_%H%M%S): Exited Grafana backup. ERROR: ${error_message}."

       tail -n10000 ${logfile} > ${logfile_tmp}
       rm ${logfile}
       mv ${logfile_tmp} ${logfile}

       exit 1
    fi
}

# Main
_initialize >> "${logfile}" 2>&1
_backup >> "${logfile}" 2>&1
_cleanup >> "${logfile}" 2>&1
_finalize >> "${logfile}" 2>&1
```
2. Create the following crontab entry with `sudo crontab -e` to run the script each day at 00:01:00: `0 2 * * * /srv/grafana-backup.sh`.
3. Verify that the crontab is correct with `sudo crontab -l` (run in the context of user 'pi').
4. Wait to the day after and check the log-file `/srv/grafana-backup.log` and backup-directory `/srv/ha-grafana/backup` so that backups are taken.

### Git for Grafana

We want to utilize Github Push to push Grafana dashboards as JSON-file.

Pre-requisities: That repository `grafana-config` is created in Github.

Perform the following:

1. If not already done, create a Github personal access token:
   - According [instructions](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token).
   - Set preferably an Expiration date (and keep note of the date and update the token accordingly).
   - Set the following scopes:
     - `repo`.
     - `gists`.
   - Copy the token.
2. Through the `File Editor` add-on, add the following to `.env` in `/srv/` on HA-server (where TOKEN is the copied token, ensure that username and repository is correct):
   `GITHUB_CONNECT_STRING=https://TOKEN@github.com/slittorin/grafana-config`
3. We do not need a .gitignore file as we want all files and directories to be pushed.
4. Through the 'SSH & Web terminal' run the following in the `/srv/ha-grafana/json` directory (change your email and name to match your Github account, and ensure that all directories you want to add to Github are present):\
   You will be asked to provide a comment at commit.
```bash
sudo git init
sudo git config user.email "you@example.com"
sudo git config user.name "Your Name"
```
5. Through the 'SSH & Web terminal' run the following in the `/config` directory (where TOKEN is the copied token, ensure that usename and repository is correct):\
   Note that git push can give error, that is nothing to trouble shoot.
```bash
sudo git remote add origin https://TOKEN@github.com/slittorin/grafana-config
sudo git push -u origin master
```
6. Through the `File Editor` add-on, add the file `/srv/grafana-git.sh` and add:
```bash
#!/bin/bash

# Inspired by: https://chowdera.com/2020/12/20201216140412674p.html
#              https://gist.github.com/crisidev/bd52bdcc7f029be2f295
#
# Purpose:
# This script extract json for all dashboards in Grafana, and triggers a git push commit.
#
# Pre-req.:
# - The Grafana server must be without login.
# - Git must be configured for the directory.
#
# Usage:
# ./grafana-git.sh HOST COMMENT
#
# HOST is the Grafana host/IP to connect to, including port.
# COMMENT is the comment to add to the push-commit.
# If empty, the default comment will be: "Minor change."

# Load environment variables (mainly secrets).
if [ -f "/srv/.env" ]; then
    export $(cat "/srv/.env" | grep -v '#' | sed 's/\r$//' | awk '/=/ {print $1}' )
fi

# Variables:
base_dir="/srv"
logfile="${base_dir}/log/grafana-git.log"
logfile_tmp="${base_dir}/log/grafana-git.tmp"
json_dir="${base_dir}/ha-grafana/json"
temp_dir="${base_dir}/ha-grafana/temp"

touch ${logfile}

# Check server.
if [ -z "$1" ]; then
    echo "ERROR. Server must be given."
    echo "$(date +%Y%m%d_%H%M%S): ERROR. Server must be given." >> ${logfile}
    exit 1
else
    HOST="$1"
fi

# Check input.
if [ -z "$2" ]
then
    no_comment=1
    COMMENT="Minor change."
else
    no_comment=0
    COMMENT="$2"
fi

_initialize() {
    cd "${base_dir}"

    echo ""
    echo "$(date +%Y%m%d_%H%M%S): Starting Grafana Backup."

    mkdir -p "${temp_dir}"
    mkdir -p "${json_dir}"
}

_grafana_backup() {
    echo "$(date +%Y%m%d_%H%M%S): Backing up..."

    cd ${temp_dir}
    rm -r ${temp_dir}/*

    # Walk through all dashboards.
    for dashboard_uid in $(curl -sS ${HOST}/api/search  | jq -r '.[] | select( .type | contains("dash-db")) | .uid') ; do
       # Retrieve the dashboard.
       dashboard_url=`echo ${HOST}/api/dashboards/uid/${dashboard_uid} | tr -d '\r'`
       dashboard_json=$(curl -sS ${dashboard_url})

       # Extract information from the json.
       dashboard_slug=$(echo ${dashboard_json} | jq -r '.meta | .slug ' | sed -r 's/[ \/]+/_/g' )
       dashboard_title=$(echo ${dashboard_json} | jq -r '.dashboard | .title' | sed -r 's/[ \/]+/_/g' )
       dashboard_version=$(echo ${dashboard_json} | jq -r '.dashboard | .version')
       dashboard_folder="$(echo ${dashboard_json} | jq -r '.meta | .folderTitle')"

       # Save the json to temp-dir
       mkdir -p ${temp_dir}/${dashboard_folder}
       echo ${dashboard_json} | jq -r {meta:.meta}+.dashboard > ${temp_dir}/${dashboard_folder}/${dashboard_slug}.json

       echo "$(date +%Y%m%d_%H%M%S): Retrieved dashboard with UID: ${dashboard_uid}, folder: ${dashboard_folder}, version ${dashboard_version}, title: ${dashboard_title}"
    done
}

_sync_files() {
    # Sync temp-dir with json-dir.
    # Ensure that .git directory is kept.
    cd ${temp_dir}
    rsync -aczvS --delete --exclude '.git' . ${json_dir}
    echo "$(date +%Y%m%d_%H%M%S): Synced temp-dir with json-dir."
}

_github_push() {
    cd ${json_dir}

    exit_code=0
    status_error=""

    # Add all in /config dir (according to .gitignore).
    echo "$(date +%Y%m%d_%H%M%S): Added all in base directory"
    git add .

    # Loop through all directories and add to git (according to .gitignore).
    for dir in */ ; do
        echo "$(date +%Y%m%d_%H%M%S): Added directory: ${dir}"
        git add "${dir}"
    done

    git status
    git_exit_code=$?
    if [ ${git_exit_code} -ne 0 ]
    then
        exit_code=1
        status_error+=" status (${git_exit_code})"
    fi

    git commit -m "${COMMENT}"
    git_exit_code=$?
    if [ ${git_exit_code} -ne 0 ]
    then
        exit_code=1
        status_error+=" commit (${git_exit_code})"
    fi

    git push origin master
    git_exit_code=$?
    if [ ${git_exit_code} -ne 0 ]
    then
        exit_code=1
        status_error+=" push (${git_exit_code})"
    fi

    # Check if error occured with git commands.
    if [ ${exit_code} -eq 0 ]
    then
        status="No error."
    else
        status="Error in: git${status_error}."
    fi
}

_finalize() {
    echo "$(date +%Y%m%d_%H%M%S): ${status}"

    tail -n10000 ${logfile} > ${logfile_tmp}
    rm ${logfile}
    mv ${logfile_tmp} ${logfile}

    exit 0
}

# Main
_initialize >> "${logfile}" 2>&1
_grafana_backup >> "${logfile}" 2>&1
_sync_files >> "${logfile}" 2>&1
_github_push >> "${logfile}" 2>&1
_finalize >> "${logfile}" 2>&1
```

## Backup of Server1 to NAS.

1. For the file `/srv/.env` add the following content, with password created on the NAS for the user `pi-backup` See [backup](https://github.com/slittorin/home-assistant-setup/blob/main/README.md#backup).\
   The files will be stored under `/server1/srv` on the NAS-share.
   ```
   NAS_BACKUP_PASSWORD=PIBACKUPPASSWORD
   ```
2. Create the following backup-script `/srv/backup-to-nas.sh` to copy all under `/srv` to NAS (remember to set `chmod ugo+x`).
```bash
#!/bin/bash

# This script backs up all files in /srv to NAS:
# - Ensure that copy on NAS is exact copy of /srv (besides unix permissions).
#
# Usage:
# ./backup-to-nas.sh

# Load environment variables (mainly secrets).
if [ -f "/srv/.env" ]; then
    export $(cat "/srv/.env" | grep -v '#' | sed 's/\r$//' | awk '/=/ {print $1}' )
fi

# Variables:
# -----------------------------------------------------------------
base_dir="/srv"
logfile="${base_dir}/log/backup-to-nas.log"
logfile_tmp="${base_dir}/log/backup-to-nas.tmp"
logfile_rsync="${base_dir}/log/backup-to-nas.rsync"
source_dir="/srv/"
mount_dir="/mnt/nas"
dest_dir="${mount_dir}/server1/srv"
nas_host="//192.168.2.10/server-backup"
nas_user="pi-backup"
error_occured=0
error_message=""

_initialize() {
    cd "${base_dir}"

    touch "${logfile}"

    mkdir -p ${mount_dir}

    echo ""
    echo "$(date +%Y%m%d_%H%M%S): Starting backup to NAS."
}

_mount() {
    echo "$(date +%Y%m%d_%H%M%S): Mount of NAS-share started."
    RESULT=`mount -t cifs -o username=${nas_user},password=${NAS_BACKUP_PASSWORD},vers=2.0 ${nas_host} ${mount_dir}`
    RESULT_CODE=$?
    if [ ${RESULT_CODE} -ne 0 ]; then
       error_occured=1
       error_message="Mount error"
       echo "$(date +%Y%m%d_%H%M%S): ERROR. ${error_message}. Exit code: ${RESULT_CODE}: ${RESULT}"
    else
       echo "$(date +%Y%m%d_%H%M%S): Mount of NAS-share performed."
    fi
}

_mkdir() {
    if [ ${error_occured} -eq 0 ]; then
       echo "$(date +%Y%m%d_%H%M%S): Creation of dir on NAS started."
       RESULT=`mkdir -p ${dest_dir}`
       RESULT_CODE=$?
       if [ ${RESULT_CODE} -ne 0 ]; then
          error_occured=1
          error_message="mkdir error"
          echo "$(date +%Y%m%d_%H%M%S): ERROR. ${error_message}. Exit code: ${RESULT_CODE}: ${RESULT}"
       else
          echo "$(date +%Y%m%d_%H%M%S): Creation of dir on NAS performed."
       fi
   fi
}

_rsync() {
    if [ ${error_occured} -eq 0 ]; then
       echo "$(date +%Y%m%d_%H%M%S): Rsync to NAS-share started."
       RESULT=`rsync -tr --delete --stats ${source_dir} ${dest_dir} > ${logfile_rsync}`
       RESULT_CODE=$?
       if [ ${RESULT_CODE} -ne 0 ]; then
          error_occured=1
          error_message="Rsync error"
          echo "$(date +%Y%m%d_%H%M%S): ERROR. ${error_message}. Exit code: ${RESULT_CODE}: ${RESULT}"
       else
          echo "$(date +%Y%m%d_%H%M%S): Rsync to NAS-share performed, output:"
          cat ${logfile_rsync} | sed '0,/^$/d' | sed '/^$/d'
       fi
   fi
}

_unmount() {
    if [ ${error_occured} -eq 0 ]; then
       echo "$(date +%Y%m%d_%H%M%S): Unmount of NAS-share started."
       RESULT=`umount ${mount_dir}`
       RESULT_CODE=$?
       if [ ${RESULT_CODE} -ne 0 ]; then
          error_occured=1
          error_message="Unmount error"
          echo "$(date +%Y%m%d_%H%M%S): ERROR. ${error_message}. Exit code: ${RESULT_CODE}: ${RESULT}"
       else
          echo "$(date +%Y%m%d_%H%M%S): Unmount of NAS-share performed."
       fi
   fi
}

_finalize() {
    if [ ${error_occured} -eq 0 ]; then
       echo "$(date +%Y%m%d_%H%M%S): Finished backup to NAS. No error."

       tail -n10000 ${logfile} > ${logfile_tmp}
       rm ${logfile}
       mv ${logfile_tmp} ${logfile}

       exit 0
    else
       echo "$(date +%Y%m%d_%H%M%S): Exited backup to NAS. ERROR: ${error_message}."

       tail -n10000 ${logfile} > ${logfile_tmp}
       rm ${logfile}
       mv ${logfile_tmp} ${logfile}

       exit 1
    fi
}

# Main
_initialize >> "${logfile}" 2>&1
_mount >> "${logfile}" 2>&1
_mkdir >> "${logfile}" 2>&1
_rsync >> "${logfile}" 2>&1
_unmount >> "${logfile}" 2>&1
_finalize >> "${logfile}" 2>&1
```
3. Create the following crontab entry with `sudo crontab -e` to run the script each day at 00:03:00: `0 3 * * * /srv/backup-to-nas.sh`.
4. Verify that the crontab is correct with `sudo crontab -l` (run in the context of user 'pi').
5. Wait to the day after and check the log-file `/srv/backup-to-nas.log` and that files are correctly copied to the NAS-share.

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
9. Setup Recorder correctly to keep data in database for 30 days, and write every 30:th (Updated 13/3-2022 from 10:th) second to the database to reduce load (even though we do not need it since we have an SSD disk instead of SD Card), and ensure that logbook is enabled:
   - Through the `File Editor` add-on, edit the file [/config/configuration.yaml](https://github.com/slittorin/home-assistant-config/blob/master/configuration.yaml) and add after `recorder:`:
     ```yaml
         purge_keep_days: 30
         auto_purge: true
         commit_interval: 30
     
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
