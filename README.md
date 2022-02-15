Docker Canvas for Integration Testing
-------------------------------

Docker provisioning for Canvas integration tests (via LTI, etc)

## Prerequisites

* [Docker Engine](https://docs.docker.com/engine/installation/)
* [Docker Compose](https://docs.docker.com/compose/install/)
* Large amount of memory allocated to your docker machine (Canvas uses a lot of memory). You need ~10GB to build the image and ~6GB to run the image.

# Setting Up

---
---
## **Note: As of 2022-02-14 and release/2022-02-16.01**

Using Canvas commit [`60fe0e86c40a346b6ff845e86b3156c7fbf5d32a`](https://github.com/instructure/canvas-lms/releases/tag/release%2F2022-02-16.01)

Refer to the [Quick Start](https://github.com/instructure/canvas-lms/wiki/Quick-Start/3b9ade72fd0cb8a98fea2241c8b8f138bb4b8286) instructions.

The `script/docker_dev_setup.sh` automated setup script is currently working
on Debian 11 with the following notes:

- [dory](https://github.com/FreedomBen/dory/tree/3420fddf14ae7cfa443612c4cc50d11073311649) appears to be mandatory. Install it on the host, not in a container.
- To reset the state of the build, remember to delete both images and volumes.
- Apply the following patch:

```diff
commit a30dd213fee92f452160065f50ade61805d87330 (HEAD)
Author: William Ono <william.ono@ubc.ca>
Date:   Mon Feb 14 16:21:39 2022 -0800

    Account for different CLI syntax

diff --git a/script/common/os/linux/impl.sh b/script/common/os/linux/impl.sh
index 6223f790..fedf0430 100644
--- a/script/common/os/linux/impl.sh
+++ b/script/common/os/linux/impl.sh
@@ -3,10 +3,10 @@ source script/common/utils/spinner.sh
 
 function set_service_util {
   if installed service; then
-    service_manager='service'
+    docker_status='service docker status'
     start_docker="sudo service docker start"
   elif installed systemctl; then
-    service_manager='systemctl'
+    docker_status='systemctl status docker'
     start_docker="sudo systemctl start docker"
   else
     echo "Unable to find 'service' or 'systemctl' installed."
@@ -15,7 +15,7 @@ function set_service_util {
 }
 
 function start_docker_daemon {
-  eval "$service_manager docker status &> /dev/null" && return 0
+  eval "$docker_status &> /dev/null" && return 0
   prompt 'The docker daemon is not running. Start it? [y/n]' confirm
   [[ ${confirm:-n} == 'y' ]] || return 1
   eval "$start_docker"
```

This repository will be retained in case the upstream procedure breaks again,
or a setup without dory is preferred.

---
---


## Clone Repo

    git clone https://github.com/ubc/docker-canvas.git docker-canvas

## Generate Canvas Docker Image (with issues encountered on stable branch as of 2021-05-18)

Based on SHA `9ad21650ebbee144bd96a28aab53507a1bcefc6c`. Still working as of tag `release\2021-06-23.26`.

The official Canvas docker image might not be up-to-date with the version available on github. If you need an updated image, you will have to build it yourself. Check out Canvas from Instructure's github (make sure you're on the branch you need, e.g.: stable). *You will also need to copy the `Dockerfile.prod.with.fixes` file into the Canvas-lms repo* and run:

    docker build -t instructure/canvas-lms:stable -f Dockerfile.prod.with.fixes .

Note that Instructure recommends at around 10 GB of RAM to build this image. This will build and tag the image as a newer version in your docker cache.

Notes:
- There is currently no Dockerfile in Canvas that will generate an easily runnable image. `Dockerfile.prod.with.fixes` is a stopgap to getting a working version of Canvas running without dory/dinghy.
- It is a combination of Canvas repo's `Dockerfile` and `ubuntu.development.Dockerfile` files.
- `yarn install` needs the `--network-timeout 600000 --network-concurrency 1` options or it will fail.`

## If it is the first time running:

Initialize data by first starting the database:

    docker compose up -d db

Wait a few moments for the database to start then (command might fail if database hasn't finished first time startup):

    CANVAS_RAILS5_2=1 docker compose run --rm app bundle exec rake db:create db:initial_setup

When prompted enter default account email, password, and display name. Also choose to share usage data or not.

The branding assets must also be manually generated when canvas is in production mode:

    docker compose run --rm app bundle exec rake canvas:compile_assets
    docker compose run --rm app bundle exec rake brand_configs:generate_and_upload_all

Edit `/ect/hosts` and add the line:

    127.0.0.1 docker_canvas_app

Finally startup all the services:

    docker compose up -d

Canvas is accessible at

    http://docker_canvas_app:8900/

MailHog (catches all out going mail from canvas) is accessible at

    http://localhost:8902/

# Running

## Start Server

    docker compose up -d

## Check Logs

    # app
    docker logs -f docker-canvas_app_1

    # more detailed app logs
    docker exec -it docker-canvas_app_1 tail -f log/production.log

    # worker
    docker logs -f docker-canvas_worker_1

    # db
    docker logs -f docker-canvas_db_1

    # redis
    docker logs -f docker-canvas_redis_1

    # mail
    docker logs -f docker-canvas_mail_1

## Stop Server

    docker compose stop

## Stop Server and Clean Up

    docker compose down
    rm -rf .data

## Update the DB

    CANVAS_RAILS5_2=1 docker compose run --rm app bundle exec rake db:migrate

## Communicating between projects

The `docker-compose.yml` is setup to allow other docker compose projects to connect via external networks. In `docker-compose.yml` you will see

```
version: '3.8'
services:
  ...
  app: &app
    ...
    networks:
      default:
        aliases:
          - app
      docker_canvas_bridge:
        aliases:
          - docker_canvas_app
  ...
networks:
  docker_canvas_bridge:
    name: docker_canvas_bridge
```

To include the network in another project you just need to add the network (in additional to the default network) and then you can use the alias `docker_canvas_app:8900` to connect to the canvas app. For example in another project do:

```
version: '3.8'
services:
  ...
  app:
    ...
    networks:
      - default
      - docker_canvas_bridge
  ...
networks:
  docker_canvas_bridge:
    external: true
```

Finally you need to edit your `/ect/hosts` and add the line (if you haven't already):

    127.0.0.1 docker_canvas_app

So your local machine can connect to Canvas using the same alias (this is important for LTI launch redirects).

# Environment Variable Configuration

## Passenger

`PASSENGER_STARTUP_TIMEOUT`: Increase to avoid first time startup Passenger timeout errors (can take a while and the timeout might be too short).

# Update postgres version (from 9.6 to 12.4)

1. `docker compose down`
1. open `docker-compose.yml` in an editor
    - comment out the regular section of `db`
    - uncomment the `tianon/postgres-upgrade` section of `db`
1. in file explorer / Finder, open the `.data` folder
    - back a backup copy of the entire `postgres` folder
    - rename the `postgres` folder to `postgres-9.6`
1. `docker compose up -d`
1. wait a few minutes for the upgrade to happen
1. `docker compose down`
1. in file explorer / Finder, open the `.data` folder
    - rename the `postgres-12` folder to `postgres`
    - edit the `pg_hba.conf` file in `postgres` and add `host all all all md5` to the bottom
1. open `docker-compose.yml` in an editor
    - uncomment the regular section of `db`
    - comment out the `tianon/postgres-upgrade` section of `db`
1. `docker compose up -d`

After you verify that the update worked, you can remove the backup copy of `postgres` and the `postgres-9.6` folders (or keep them as backups for later)
