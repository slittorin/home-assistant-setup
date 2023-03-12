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
  - [Git for shell-scripts](https://github.com/slittorin/home-assistant-setup#git-for-shell-scripts)
  - [OS/HW Statistics](https://github.com/slittorin/home-assistant-setup#oshw-statistics)
  - [Docker volume sizes](https://github.com/slittorin/home-assistant-setup/blob/main/README.md#docker-volume-sizes).
  - [Installation for InfluxDB](https://github.com/slittorin/home-assistant-setup#installation-for-influxdb)
    - [Backup for InfluxDB](https://github.com/slittorin/home-assistant-setup#backup-for-influxdb)
    - [Export for InfluxDB](https://github.com/slittorin/home-assistant-setup#export-for-influxdb)
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
3. Through the `File Editor` add-on, create [/srv/.gitignore](https://github.com/slittorin/server1-config/blob/master/.gitignore).
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
   - Create the file [/srv/os-stats.sh](https://github.com/slittorin/server1-config/blob/master/os-stats.sh).
2. Make it executable with `chmod ug+x os-stats.sh`.
3. Add it to crontab with `sudo crontab -e` to run every 15 minutes by adding: `*/15 * * * * /srv/os-stats.sh`
4. Check with `sudo crontab -l` that the row was added.

## Docker volume sizes

We want to track docker volume size statistics, that can be pulled into HA.

1. We create a script that creates the necessary docker volume size statisticsls that we can store in files, that can be read by Home Assistant.
   - Create the  file [/srv/docker-volume-sizes.sh](https://github.com/slittorin/server1-config/blob/master/docker-volume-sizes.sh).
2. Make it executable with `chmod ug+x docker-volume-sizes.sh`.
3. Add it to crontab with `sudo crontab -e` to run once each hour by adding: `0 * * * * /srv/docker-volume-sizes.sh`
4. Check with `sudo crontab -l` that the row was added.

## Installation for InfluxDB

1. Check versions of available docker images for InfluxDB at [Docker - InfluxDB](https://hub.docker.com/_/influxdb).
   - If you do not want the 'latest' version, use version number.
   - At time of writing (20220207) the 'latest' version is 2.1.1 (isolated with `sudo docker image inspect influxdb` and looking for 'INFLUXDB_VERSION').
2. Create the directory `/srv/ha-history-db`, and the following sub-directories:
   - `backup`.
   - `export`.
   - `import`.
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
4. For the  file [/srv/docker-compose.yml](https://github.com/slittorin/server1-config/blob/master/docker-compose.yml) add the concent necessary for ha-history-db container.
5. In the `/srv` directory:
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
6. In a web browser go the IP address (or hostname) of server1 and port 8086, for example [http://192.168.2.30:8086/](http://192.168.2.30:8086/).
   - Through 'Data -> API Tokens -> admin's Token', copy the token and add to `HA_HISTORY_DB_ROOT_TOKEN` in `/srv/.env`.
   -  Go to `Data` -> `API Tokens` -> `Generate API Token` -> `Read/Write API Token`:
      - Set a descriptive name: `Read to HA bucket`
      - Choose bucket `ha` for read, but remove from write.
      - Choose the newly created token and copy the token and add to `HA_HISTORY_DB_GRAFANA_TOKEN` in `/srv/.env`.

### Backup for InfluxDB

1. Create the backup-script [/srv/influxdb-backup.sh](https://github.com/slittorin/server1-config/blob/master/influxdb-backup.sh) to take InfluxDB-backup through docker-compose (remember to set `chmod ugo+x`).
2. Create the following crontab entry with `sudo crontab -e` to run the script each day at 01:00:00: `* 1 * * * /srv/influxdb-backup.sh`.
3. Verify that the crontab is correct with `sudo crontab -l` (run in the context of user 'pi').
4. Wait to the day after and check the log-file `/srv/log/influxdb-backup.log` and backup-directory `/srv/ha-history-db/backup` so that backups are taken.

### Export for InfluxDB

After the [SSD-failed](https://github.com/slittorin/home-assistant-maintenance/blob/main/README.md#failed-ssd-drive) in early 2023, and that restore of the InfluxDB-database gave a week of data that was lost slightly after summer 2022, I realised I needed a second way to backup my data.

Here I export each days data into csv-file, so that the data can be imported if necessary.

1. Create the export-script [/srv/influxdb-export-yesterday.sh](https://github.com/slittorin/server1-config/blob/master/influxdb-export-yesterday.sh) (remember to set `chmod ugo+x`).
2. Create the following crontab entry with `sudo crontab -e` to run the script each day at 00:10:00: `10 0 * * * /srv/influxdb-export-yesterday.sh`.
3. Verify that the crontab is correct with `sudo crontab -l` (run in the context of user 'pi').
4. Wait to the day after and check the log-file `/srv/log/influxdb-export-yesterday.log` and export-directory `/srv/ha-history-db/export` so that backups are taken.

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
4. For the following file [/srv/docker-compose.yml](https://github.com/slittorin/server1-config/blob/master/docker-compose.yml) add the content necessary for ha-grafana container.
5. In the `/srv` directory:
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
6. In web browser go the IP address (or hostname) of server1 and port 3000, for example [http://192.168.2.30:3000/](http://192.168.2.30:3000/) (yes, we setup grafana to utilize no credentials).
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

1. Create the backup-script [/srv/grafana-backup.sh](https://github.com/slittorin/server1-config/blob/master/grafana-backup.sh) to take Grafana-backup of the Sqlite-database file (remember to set `chmod ugo+x`).
2. Create the following crontab entry with `sudo crontab -e` to run the script each day at 02:00:00: `0 2 * * * /srv/grafana-backup.sh`.
3. Verify that the crontab is correct with `sudo crontab -l` (run in the context of user 'pi').
4. Wait to the day after and check the log-file `/srv/log/grafana-backup.log` and backup-directory `/srv/ha-grafana/backup` so that backups are taken.

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
4. Run the following in the `/srv/ha-grafana/json` directory (change your email and name to match your Github account, and ensure that all directories you want to add to Github are present):\
   You will be asked to provide a comment at commit.
```bash
sudo git init
sudo git config user.email "you@example.com"
sudo git config user.name "Your Name"
```
5. Run the following in the `/srv/ha-grafana/json` directory (where TOKEN is the copied token, ensure that usename and repository is correct):\
   Note that git push can give error, that is nothing to trouble shoot.
```bash
sudo git remote add origin https://TOKEN@github.com/slittorin/grafana-config
sudo git push -u origin master
```
6. Create the file [/srv/grafana-git.sh](https://github.com/slittorin/server1-config/blob/master/server1-git.sh).

## Backup of Server1 to NAS.

1. For the file `/srv/.env` add the following content, with password created on the NAS for the user `pi-backup` See [backup](https://github.com/slittorin/home-assistant-setup/blob/main/README.md#backup).\
   The files will be stored under `/server1/srv` on the NAS-share.
   ```
   NAS_BACKUP_PASSWORD=PIBACKUPPASSWORD
   ```
2. Create the backup-script [/srv/backup-to-nas.sh](https://github.com/slittorin/server1-config/blob/master/backup-to-nas.sh) to copy all under `/srv` to NAS (remember to set `chmod ugo+x`).
3. Create the following crontab entry with `sudo crontab -e` to run the script each day at 03:00:00: `0 3 * * * /srv/backup-to-nas.sh`.
4. Verify that the crontab is correct with `sudo crontab -l` (run in the context of user 'pi').
5. Wait to the day after and check the log-file `/srv/log/backup-to-nas.log` and that files are correctly copied to the NAS-share.

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
3. Through the `File Editor` add-on, edit the file [/config/configuration.yaml](https://github.com/slittorin/home-assistant-config/blob/master/configuration.yaml) and add `recorder:` configuration.
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
6. Through the `File Editor` add-on, edit the file [/config/configuration.yaml](https://github.com/slittorin/home-assistant-config/blob/master/configuration.yaml) and:
   - Add at `allowlist_external_dirs:` and `packages:` configuration to bottom of the file.
   - Add `sensor: !include sensors.yaml` configuration.
   - Add `logger:` configuration.
   - Add `purge_keep_days:`, `auto_purge:` and `commit_interval:` configuration.
7. Add areas that is representative for your home.
   - Go to `Configuration` -> `Devices and services` -> `Areas` and update the rooms/areas that represent your home.
8. Verify the config-file and restart.
9. Verify that the setup is working correct by looking in the Home Assistant logfile.

## History DB setup

1. In a web browser go the IP address (or hostname) of server1 and port 8086, for example [http://192.168.2.30:8086/](http://192.168.2.30:8086/).
   - Login with the username and password setup above.
   - Go to `Data` -> `API Tokens` -> `Generate API Token` -> `Read/Write API Token`:
     - Set a descriptive name: `Read/Write to HA bucket`
     - Choose bucket `ha` for both read and write.
   - Choose the newly created token and copy the token.
2. Through the `File Editor` add-on, edit the file `/config/secrets.yaml` and add (change the string 'token' below to the right token):
   `history_db_token: TOKEN`.
3. Through the `File Editor` add-on, edit the file [/config/configuration.yaml](https://github.com/slittorin/home-assistant-config/blob/master/configuration.yaml) and add configuration for history and influxdb.
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
