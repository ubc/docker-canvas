Docker Canvas for Integration Testing
-------------------------------

Docker provisioning for Canvas integration tests (via LTI, etc)

### Prerequisites

* [Docker Engine](https://docs.docker.com/engine/installation/)
* [Docker Compose](https://docs.docker.com/compose/install/)

### Clone Repo and Start Server

    git clone https://github.com/ubc/docker-canvas.git docker-canvas

### If it is the first time:

Initialize data by first starting the database:

    docker-compose up -d db

Wait a few moments for the database to start then (command might fail if database hasn't finished first time startup):

    docker-compose run --rm app bundle exec rake db:create db:initial_setup

The branding assets must also be manually generated when canvas is in production mode:

    docker-compose run --rm app bundle exec rake brand_configs:generate_and_upload_all

When prompted enter default account email, password, and display name. Also choose to share usage data or not.

Finally startup all the services:

    docker-compose up -d

Canvas is accessible at

    http://localhost:8900/

MailHog (catches all out going mail from canvas) is accessible at

    http://localhost:8901/

### Start Server

    docker-compose up -d

### Check Logs

    # app
    docker logs -f dockercanvas_app_1
    # worker
    docker logs -f dockercanvas_worker_1
    # db
    docker logs -f dockercanvas_db_1
    # redis
    docker logs -f dockercanvas_redis_1
    # mail
    docker logs -f dockercanvas_mail_1

### Stop Server

    docker-compose stop

### Stop Server and Clean Up

    docker-compose down
    rm -rf .data

### Enable Virtual Hosts

 It may be hard to link to the Canvas container in some situations using only localhost. This can be mitigated using the IP address of your host machine to access the canvas instance or by using virtual hosts if that is not feasible. One way of setting up virtual hosts in docker is by using [Dory](https://github.com/FreedomBen/dory).

    gem install dory
    dory up

Now you can access the canvas via `canvas.docker`. Adding a `docker-compose.override.yml` file to other projects can also improve communication between docker contains. See [docker-compose.example.override.yml](docker-compose.example.override.yml) for an example of how this can be done. Docker compose will automatically use `docker-compose.override.yml` is present when starting your containers.
