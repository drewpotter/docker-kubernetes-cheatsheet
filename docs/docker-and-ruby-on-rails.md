# Docker and Ruby on Rails

When using Docker typically we will have a `docker-compose.yml` file which will define our application using a set of containers.

## Docker Compose
Here is an example `docker-compose.yml` with explanations, please note that the spacing of each line is important:

```
version: '3'
services:
  web:
    build: .
    command: bundle exec rails s -p 3000 -b '0.0.0.0'
    volumes:
      - .:/ProtectedPlanet
      - protectedplanet_import_data:/import_data
    ports:
      - "3000:3000"
    env_file:
      - '.env'
    depends_on:
      - db
      - redis
      - elasticsearch

  db:
    container_name: protectedplanet-db
    image: kartoza/postgis
    ports:
      - "5432:5432"
    env_file:
      - '.env'
    volumes:
      - protectedplanet_pg_data:/var/lib/postgresql

  redis:
    image: redis
    volumes:
      - protectedplanet_redis_data:/data

  sidekiq:
     build: .
     volumes:
       - .:/ProtectedPlanet
       - protectedplanet_import_data:/import_data
     links:
       - db
       - redis
     command: bundle exec sidekiq
     env_file:
       - '.env'

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:5.4.3
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    ports:
      - "9200:9200"
    env_file:
      - '.env'

  kibana:
    image: docker.elastic.co/kibana/kibana:5.4.3
    env_file:
      - '.env'
    ports:
      - "5601:5601"

volumes:
  protectedplanet_pg_data:
    driver: local
  protectedplanet_redis_data:
    driver: local
  protectedplanet_import_data:
    driver: local
```

First we define the Docker Compose version, which is set to version 3. The reason for this is that the different versions of Docker can have different new features or potentially removed features. So we need to define the version which we are going to use.

### Services
Next we define the `services`.

#### The Rails application
We use the identifier `web` for the Ruby on Rails application itself.

We use `build: .` to instruct Docker to build the current directory using the corresponding Dockerfile, which we will discuss later.
We use `command: bundle exec rails s -p 3000 -b '0.0.0.0'` to instruct Docker to run this familiar command when we bring up the application with Docker Compose.

We use `volumes` to instruct Docker to make accessible certain folders or volumes to the `web` docker container.

In this case, we mount the current directory of the project signified by `.` as `/ProtectedPlanet` inside the Docker container.

Also we mount the Docker volume called `protectedplanet_import_data` as `/import_data`.

The reason for this is that when the Docker container gets rebuilt or destroyed, we can persist the data inside a Docker volume so that we can carry on with the same data as before.

We use `ports` to specify the port to expose both on the host machine and the Docker container. In this case we set them both to `3000`, but you can specify different ports for host or Docker container if you want to simply by changing this number. `3000` is the default Rails port.

We are using `env_file` to reference the `.env` file. Although this will be removed as we switch this guide over to using Rails encrypted secrets.

Finally for this `web` service we list our dependencies. We use `depends_on` to state that we must have the `db`, `redis` and `elasticsearch` services running before we start the `web` service.

#### The database
The next service we need is `db` for our database.

We specify the container name to be `protectedplanet-db` this helps if we have other Dockerised projects which might have the same name of simply `db`.

The `image` is specified as `kartoza/postgis` which is a pre-built Docker image which includes support for PostGIS.

The `ports` which are exposed both on the host machine and the Docker container are `5432`.

Again we refer to the `env_file` which will be removed and replaced with Ruby on Rails encrypted secrets.

Finally we create `volumes` with a special Docker volume for storing persistent database data which will not get deleted when the Docker container is destroyed or updated.

#### Redis
The next service we define is `redis` using an `image` called `redis`.

We also create a Docker volume called `protectedplanet_redis_data` which we mount at the `/data` path inside the Docker container.

#### Sidekiq
For Sidekiq we build the same image as we do for the `web` service, which is the current working directory, using the Dockerfile which we will look at later on.

We also mount the necessary volumes which are the current working directory `.` as `/ProtectedPlanet` and the `protectedplanet_import_data` as `/import`. This is so that the Sidekiq jobs will be able to access everything they need.

We use `links` to link the `db` and `redis` containers to this `sidekiq` container.

The `command` of `bundle exec sidekiq` is used to load up sidekiq inside the container.

Finally we again have `env_file` which will be removed once we have switched to using Rails encrypted secrets.
