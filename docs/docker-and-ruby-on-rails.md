# Docker and Ruby on Rails

Let us take a look at the Dockerisation of ProtectedPlanet as an example.

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

### Docker Compose services
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

#### Elasticsearch
For Elasticsearch we define the service `elasticsearch` and use the image `docker.elastic.co/elasticsearch/elasticsearch` and we specify the version with the colon followed by the version.

Next we set some environment variables which are from the Elasticsearch docs as well as the `ulimits` which are beyond the scope of this cheatsheet.

We open up the port `9200` on the local machine and the Docker container.

Again we use the `env_file` which will be removed at some point.

#### kibana
Kibana is a way to monitor Elasticsearch. I thought this would be interesting to monitor what was going on.

We define a service `kibana` and use the image from `docker.elastic.co/kibana/kibana`
and again we specify the version using the colon followed by the version number.

Finally we have the `ports` which are `5601` on both the host machine and the Docker container.


### Docker Compose volumes
Finally at the end of the `docker-compose.yml` we have the three Docker volumes which we make persistent. This means that the data will survive the Docker containers being destroyed or updated.

Here we have the following volumes: `protectedplanet_pg_data`, `protectedplanet_redis_data` and `protectedplanet_import_data`.


## The Dockerfile
Let us now take a look at the Dockerfile for this project, with a detailed overview.

```
FROM ruby:2.3
MAINTAINER andrew.potter@unep-wcmc.org

RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    git

RUN apt-get install software-properties-common -y

WORKDIR /gdal
RUN wget http://download.osgeo.org/gdal/1.11.5/gdal-1.11.5.tar.gz
RUN tar -xvf gdal-1.11.5.tar.gz
RUN cd gdal-1.11.5 \
    && ./configure --prefix=/usr \
    && make \
    && make install

WORKDIR /postgres
RUN wget https://ftp.postgresql.org/pub/source/v11.1/postgresql-11.1.tar.gz
RUN tar -xvf postgresql-11.1.tar.gz
RUN cd postgresql-11.1 \
    && ./configure --prefix=/usr \
    && make \
    && make install

WORKDIR /node
RUN wget http://nodejs.org/dist/v10.8.0/node-v10.8.0.tar.gz
RUN tar -xvf node-v10.8.0.tar.gz
RUN ls node-v10.8.0
RUN cd node-v10.8.0 \
    && ./configure --prefix=/usr \
    && make install \
    && wget https://www.npmjs.org/install.sh | sh

RUN whereis npm
RUN npm install bower -g

WORKDIR /geos
RUN wget https://download.osgeo.org/geos/geos-3.7.0.tar.bz2
RUN tar -xvf geos-3.7.0.tar.bz2
RUN ls geos-3.7.0
RUN cd geos-3.7.0 \
    && ./configure --prefix=/usr \
    && make install

WORKDIR /ProtectedPlanet
ADD Gemfile /ProtectedPlanet/Gemfile
ADD Gemfile.lock /ProtectedPlanet/Gemfile.lock
RUN gem install rgeo --version '=0.4.0' -- --with-geos-dir=/usr/lib
RUN bundle install

ARG USER=node
ARG UID=1000
ARG HOME=/home/$USER
RUN adduser --uid $UID --shell /bin/bash --home $HOME $USER

COPY . /ProtectedPlanet

EXPOSE 3000

CMD ["rails", "server", "-b", "0.0.0.0"]
```

TODO: Add details.
