# Docker and Ruby on Rails

When using Docker typically we will have a `docker-compose.yml` file which will define our application using a set of containers.

Here is an example `docker-compose.yml` with explanation:

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

Next we define the `services`. 

We use the identifier `web` for the Ruby on Rails application itself.

We use `build: .` to instruct Docker to build the current directory using the corresponding Dockerfile, which we will discuss later.
We use `command: bundle exec rails s -p 3000 -b '0.0.0.0'` to instruct Docker to run this familiar command when we bring up the application with Docker Compose.

We use `volumes` to instruct Docker to make accessible certain folders or volumes to the `web` docker container.