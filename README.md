Docker Canvas for Integration Testing
-------------------------------

Docker provisioning for Canvas integration tests (via LTI, etc)

### Prerequisites

* [Docker Engine](https://docs.docker.com/engine/installation/)
* [Docker Compose](https://docs.docker.com/compose/install/)
* Large amount (~4GB) of memory allocated to your docker machine (Canvas uses a lot of memory)

### Clone Repo and Start Server

    git clone https://github.com/ubc/docker-canvas.git docker-canvas

### If it is the first time:

Initialize data by first starting the database:

    docker-compose up -d db

Wait a few moments for the database to start then (command might fail if database hasn't finished first time startup):

    docker-compose run --rm app bundle exec rake db:create db:initial_setup


When prompted enter default account email, password, and display name. Also choose to share usage data or not.

The branding assets must also be manually generated when canvas is in production mode:

    docker-compose run --rm app bundle exec rake \
        canvas:compile_assets_dev \
        brand_configs:generate_and_upload_all

Finally startup all the services (the build will create a docker image for you):

    docker-compose up -d --build

Canvas is accessible at

    http://localhost:8900/

MailHog (catches all out going mail from canvas) is accessible at

    http://localhost:8901/

### Start Server

    docker-compose up -d

### Check Logs

    # app
    docker logs -f docker-canvas_app_1
    # worker
    docker logs -f docker-canvas_worker_1
    # db
    docker logs -f docker-canvas_db_1
    # redis
    docker logs -f docker-canvas_redis_1
    # mail
    docker logs -f docker-canvas_mail_1

### Stop Server

    docker-compose stop

### Stop Server and Clean Up

    docker-compose down
    rm -rf .data

### Rebuild local image

You can try rebuilding the image if you are experiencing issues importing course content, etc. Before running this command, stop the server (if it's running) using `docker-compose down`

    docker-compose build

### Communicating between projects

 It may be hard to link to the Canvas container in some situations using only `localhost`. This can be mitigated using the IP address of your host machine to access the canvas instance or by using virtual hosts if that is not feasible.

 Better instructions on getting this working will be add in the future (sorry!).
