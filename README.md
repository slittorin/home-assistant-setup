# Home Assistant - Installation

## Prequisities

- We want a clean and module-based installation.
- We utilize MariaDB as main database-engine.
- We utilize InfluxDB as history database-engine.
- Setup a RPI instance with [Raspberry PI install](https://github.com/slittorin/raspberrypi-install/).

## Installation for MariaDB

1. Create the directory `/srv/ha-db`.
2. Create the following file `/src/docker-compose.yml` with the following content:
    ```
    version: '3'
    services:
      db:
    ```
3. 
