# docker-101

My goal is to learn a better way of developing, deploying, and testing my applications without the headache of getting my local environment switched around each time I jump into different projects. I also aim to better understand CI/CD.

## Docker

Docker is an open platform for developing, shipping, and running applications. It allows you to separate your applications from the infrastructure. Docker follows a client-server architecture.

- **Documents**: [Docker Installation Guide for Ubuntu](https://docs.docker.com/desktop/install/ubuntu/)

## Definitions

- **Docker**: Docker is a way to package software so it can run on any hardware.
- **Dockerfile**: A blueprint for building a Docker Image.
- **Image**: A read-only template with instructions for creating a Docker Container.
- **Container**: A running instance of a Docker Image/process, such as a Node application or Ruby application.

## Docker File Format

Each instruction in the Dockerfile is treated as a layer.

- **FROM**: Defines a base for your image.
- **RUN**: Executes a command in a new layer on top of the current image.
- **WORKDIR**: Sets the working directory for any command executed in the Dockerfile.
- **COPY**: Copies new files or directories from `<src>` to the container's filesystem at `<dest>`.
- **CMD**: Specifies the default program that runs when you start the container image.

Use the name `Dockerfile` without an extension. This default name allows you to simplify the command `docker build`.

```Dockerfile
# Below is a parse directive; include this comment.
# This tells Docker what syntax to use when parsing the Dockerfile.

# syntax=docker/dockerfile:1
FROM ubuntu:22.04

# Install app dependencies
RUN apt-get update && apt-get install -y python3 python3-pip
RUN pip install flask==2.1.*

# Install app
COPY hello.py /

# Final configuration
ENV FLASK_APP=hello
EXPOSE 8000
CMD flask run --host 0.0.0.0 --port 8000
```

## Example of Usage

Let's say you build a Ruby app with some dependencies. You pass it off to a co-worker who has a different operating system version and a different version of Ruby installed. This can result in the application failing to build and run on their machine.

Docker solves this issue by packaging software so that it can run on any hardware.

# Commands

- **Start Docker-Desktop CLI**: `systemctl --user start docker-desktop`
- **Quit Docker-Desktop**: `systemctl --user stop docker-desktop`
- **Upgrade Docker-Desktop**: `sudo apt-get install ./docker-desktop-<version>-<arch>.deb`
- **Check Binary Versions**: `docker compose version`
- **Docker Version**: `docker --version`
- **Docker Version**: `docker version`

## CLI COMMANDS

- **docker ps**
- **Add Image Tag**: `docker tag docker-node:latest node-docker:v1.0.0`
- **Remove Tag**: `docker rmi docker-node:v1.0.0`
- **Run Image**: `docker run docker-node`
- **Port Forwarding**: `docker run --publish 8000:8000 docker-node` -> `docker run -p <local_port>:<container_port> <image>`
- **Detached Mode**: `docker run -d -p 8000:8000 docker-node`
- **Stop Container**: `docker stop <container_name>` -> Note that the name is randomly generated.
- **See All Containers**: `docker ps -a`
- **Remove Container**: `docker rm <container_name>`
- **Name Container**: `docker run -d -p 8000:8000 --name <container-name> <image_name>`
- **prune dangling images**: `docker image prune`
- **prune stopped containers**: `docker container prune`
- **view volumes**: `docker volume ls`
- **compose up**: `docker compose -f docker-compose.dev.yml up --build`
- **compose down**: `docker compose -f docker-compose.dev.yml down --rmi all`
  **Testing**
- **build image with testing**: `docker build -t node-docker --target test .`

## Dockerfile

- Contains code to build your Docker image.
- Runs your app as a Docker container.

## Setup

Each of the following steps is considered a "layer". Docker will try to cache these if nothing has changed.

1. Start with a `Dockerfile` in the root of the app.

   1. Set the **baseImage** with the `FROM` keyword. It will be something like `FROM NODE:<version>`.
   2. Add the app source code using `WORKDIR`. Think of this as changing directory (`CD`) into a directory.
   3. Set up dependencies to be cached. Use `COPY package*.json ./`.
   4. Install the dependencies in the container with `RUN npm install`.
   5. Copy from `<local>` to `<container>` using `COPY . .`.
   6. Set environment variables with `ENV`. For example, `ENV PORT=8080`.
   7. Expose port 8080 with `EXPOSE 8080`.
   8. Execute the command with `CMD ["npm", "start"]`.

2. Build the Docker image.

   1. Run `docker build -t ndamico28/demoapp:1.0`.

3. Run the container.
   1. With CMD: `docker run <id>`.
   2. Port forward the selected port from the Dockerfile: `docker run -p <local_port>:<docker_port> <id>`.
   3. Devj

## Development Setup

### Setup Database

Instead of downloading a database, installing, configuring and then running the database as a service, we can
use Docker offical images for a Database and run it in the container.

[Use Volumes](https://docs.docker.com/storage/volumes/)

- Create a volume for **data**.
- Create one volume for **configuration**.

```
docker volume create mongodb
docker volume create mongodb_config

# Create network so application and database can talk with each other.
docker network create mongodb

docker run -it --rm -d -v mongodb:/data/db \
  -v mongodb_config:/data/configdb -p 27017:27017 \
  --network mongodb \
  --name mongodb \
  mongo
```

## Use Compose for local development

This Compose file is super convenient as we do not have to type all the parameters to pass to the docker run command. We can declaratively do that in the Compose file.

- exposing port 9229 allows for a remote debugger to be attached.
- Setup yml file.
- Run docker compose with yml configuration: `docker compose -f docker-compose.dev.yml up --build`

```ymlversion: '3.8'
services:
 notes:
  build:
   context: .
  ports:
   - 8000:8000
   - 9229:9229
  environment:
   - SERVER_PORT=8000
   - CONNECTIONSTRING=mongodb://mongo:27017/notes
  volumes:
   - ./:/app
   - /app/node_modules
  command: npm run debug

 mongo:
  image: mongo:4.2.8
  ports:
   - 27017:27017
  volumes:
   - mongodb:/data/db
   - mongodb_config:/data/configdb
volumes:
 mongodb:
 mongodb_config:

```

## Multi-stage Dockerfile for testing

This approach will allow use to build our image with a mult-step process. It will run our tests and build
out production image.

```dockerfile
# syntax=docker/dockerfile:1
FROM node:18-alpine as base

WORKDIR /code

COPY package.json package.json
COPY package-lock.json package-lock.json

FROM base as test
RUN npm ci
COPY . .
CMD ["npm", "run", "test"]

FROM base as prod
RUN npm ci --production
COPY . .
CMD ["node", "server.js"]

```
