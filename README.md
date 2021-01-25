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

    docker-compose run --rm app bundle exec rake canvas:compile_assets
    docker-compose run --rm app bundle exec rake brand_configs:generate_and_upload_all


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

### Stop Server

    docker-compose stop

### Stop Server and Clean Up

    docker-compose down
    rm -rf .data

### Rebuild local image

You can try rebuilding the image if you are experiencing issues importing course content, etc. Before running this command, stop the server (if it's running) using `docker-compose down`

    docker-compose build

### Update the DB

    docker-compose run --rm app bundle exec rake db:migrate

### Update Canvas

The official Canvas docker image might not be up-to-date with the version available on github. If you need an updated image, you will have to build it yourself. Check out Canvas from Instructure's github. You will probably have issues building the image from the branches (master, stable, etc), so checkout out one of the release tags, e.g.:

    git checkout tags/release/2020-12-16.47

Then build the docker image with:

    docker build -t instructure/canvas-lms:stable .

Note that Instructure recommends at least 8 GB of RAM to build this image. This will build and tag the image as a newer version in your docker cache. You will need to rebuild the app and worker images to incorporate this new image:

    docker-compose build app worker

You might also need to update the DB and rebuild assets. After this, you should be able to start Canvas as usual.

#### Canvas Build Issues

##### Issues encountered on stable branch as of 2020-11-10

* Yarn install fails with `There appears to be trouble with your network connection. Retrying...` messages.

  If your network isn't having issues, then it could just be the timeout before retry is too short. I'm not sure if this is due to packages being too large or if there's some sort of throttling in effect when downloading packages. Open the Canvas `Dockerfile` and add the options `--network-timeout 600000 --network-concurrency 1` to `yarn install`. This will increase the timeout to 10 minutes (in milliseconds) and limit the download to only one at a time.

* Yarn install fails with package not found. However, when you check against the yarn registry, you can clearly see it listed.

  Remove the `node_modules` directory entirely and run the build again.

* Migration fails with: `` NoMethodError: undefined method `block_stranded' for #<Switchman::Shard:0x00005582f698fb60> ``

  Looks like db:migrate needs to run differently if you're using Rails 5.2 vs Rails 6. It defaults to the Rails 6 version. Since we're currently using Rails 5.2, we need to set environment variable CANVAS_RAILS5_2 to 1, e.g.: `CANVAS_RAILS5_2=1`

* Migration fails on MakeRoleRootAccountIdNotNull

  Edit `db/migrate/20200910165057_make_role_root_account_id_not_null.rb`:

  ```diff
  tag :predeploy

  def up
  - Role.where(:root_account_id => nil).delete_all
  + Role.where(:root_account_id => nil).update_all(:root_account_id => 0)
    change_column_null :roles, :root_account_id, false
  end

  ```

  This was a bug that was fixed by Canvas in a later commit that hasn't made it into stable yet.

### Communicating between projects

It may be hard to link to the Canvas container in some situations using only `localhost`. This can be mitigated using the IP address of your host machine to access the canvas instance or by using virtual hosts if that is not feasible.

Better instructions on getting this working will be add in the future (sorry!).

#### LTI services (membership, grades, etc)

For LTI you need to use your machine's IP address (Change the `DOMAIN` environment variable to your IP address).

### Troubleshooting

##### Passenger timeout when trying to access Canvas.

Increase the PASSENGER_STARTUP_TIMEOUT environment variable in docker-compose.yml. First time startup can take a while and the timeout might be too short.

### Update postgres version (from 9.6 to 12.4)

1. `docker-compose down`
1. open `docker-compose.yml` in an editor
    - comment out the regular section of `db`
    - uncomment the `tianon/postgres-upgrade` section of `db`
1. in file explorer / Finder, open the `.data` folder
    - back a backup copy of the entire `postgres` folder
    - rename the `postgres` folder to `postgres-9.6`
1. `docker-compose up -d`
1. wait a few minutes for the upgrade to happen
1. `docker-compose down`
1. in file explorer / Finder, open the `.data` folder
    - rename the `postgres-12` folder to `postgres`
    - edit the `pg_hba.conf` file in `postgres` and add `host all all all md5` to the bottom
1. open `docker-compose.yml` in an editor
    - uncomment the regular section of `db`
    - comment out the `tianon/postgres-upgrade` section of `db`
1. `docker-compose up -d`


After you verify that the update worked, you can remove the backup copy of `postgres` and the `postgres-9.6` folders (or keep them as backups for later)
