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
4. Create the directory `/srv/ha-db`.
5. Create the following file `/src/docker-compose.yml` with the following content:
```
version: '3'
services:
  ha-db:
    image: mariadb
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: changeme
      MYSQL_DATABASE: ha-db
      MYSQL_USER: ha
      MYSQL_PASSWORD: [not shown here]
    logging:
      driver: syslog
    networks:
      - backend
    volumes:
      - /srv/ha-db:/var/lib/mysql
```
3. docker-compose pull
