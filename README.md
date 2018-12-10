# cyb3rwr3ck/openfire

- [Introduction](#introduction)
- [Getting started](#getting-started)
  - [Installation](#installation)
  - [Quickstart](#quickstart)
- [Docker Compose](#docker-compose)
- [Persistence](#persistence)
- [Logs](#logs)
- [References](#references)

# Introduction

[Dockerfile](openfire/app/Dockerfile) to create a [Docker](https://www.docker.com/) container image for [Openfire](http://www.igniterealtime.org/projects/openfire/).

Openfire is a real time collaboration (RTC) server licensed under the Open Source Apache License. It uses the only widely adopted open protocol for instant messaging, XMPP (also called Jabber). Openfire is incredibly easy to setup and administer, but offers rock-solid security and performance.

This project is based on work by [sameersbn/openfire](/https://github.com/sameersbn/docker-openfire) and [gizmotronic/docker-openfire](https://github.com/gizmotronic/docker-openfire).

# Getting started

## Installation

At the moment we do not maintain an automated build of the image. You'll have to build it yourself doing

```bash
docker build -t openfire .
```

from the [`./openfire/app/`](openfire/app/) directory (thats the one containing the Dockerfile)

## Quickstart

Then you can run a container like this:

```bash
docker run --name openfire -d --restart=always \
  --publish 9090:9090 --publish 5222:5222 --publish 7777:7777 \
  --volume /srv/docker/openfire:/var/lib/openfire \
  openfire:latest
```

# Docker Compose

The main changes compared to the projects this one is derived from is the integration in a custom docker-compose environment.

We also rely on external [jwilder/nginx-proxy](https://github.com/jwilder/nginx-proxy) and [JrCs/docker-letsencrypt-nginx-proxy-companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion) containers that are maintained outside of this project as a seperate docker-compose setup.

The [docker-compose.yml](docker-compose.yml) file is used to set up the `openfire` container using an external `mysql` container instead of using the embedded-db. (The option is still there, though...)

First edit the [openfire.conf](openfire.conf) file (which is symlinked as [.env](.env) [Environment file](https://docs.docker.com/compose/env-file/)) and set your desired domain name. Leave the data dir as is, it's the same that ist configured inside the [Dockerfile](openfire/app/Dockerfile) and [openfire/app/entrypoint.sh](openfire/app/entrypoint.sh) Shell Script.

```ApacheConf
OPENFIRE_HOSTNAME=jabber.example.com
OPENFIRE_DATA_DIR=/var/lib/openfire
```

then do

```bash
docker-compose up -d
```

Or for testing purposes stay attached to all containers by ommiting the `-d` flag.

Point your browser to http://localhost:9090 and follow the setup procedure to complete the installation. The [Build A Free Jabber Server In 10 Minutes](https://www.youtube.com/watch?v=ytUB5qJm5HE#t=246s) video by HAKK5 should help you with the configuration and also introduce you to some of its features.

When using the `mysql` database instead of the embedded one just use the container name `openfire-db:3600` to provide openfire with the db.

## Persistence

For Openfire to preserve its state across container shutdown and startup you should mount a volume at `/var/lib/openfire`.

> *The [Quickstart](#quickstart) command already mounts a volume for persistence.*

SELinux users should update the security context of the host mountpoint so that it plays nicely with Docker:

```bash
mkdir -p /srv/docker/openfire
chcon -Rt svirt_sandbox_file_t /srv/docker/openfire
```

## Java VM options

You may append options to the startup command to configure the JVM:

```bash
docker run -name openfire -d \
  [DOCKER_OPTIONS] \
  gizmotronic/openfire:4.2.3 \
  -XX:+UseConcMarkSweepGC -XX:+CMSIncrementalMode
```

## Logs

To access the Openfire logs, located at `/var/log/openfire`, you can use `docker exec`. For example, if you want to tail the logs:

```bash
docker exec -it openfire tail -f /var/log/openfire/info.log
```

# Maintenance

## Upgrading

When a new version of openfire is released and you have setup persistence correctly just do the following from the project directory containing the [docker-compose.yml](docker-compose.yml):

1. stop and rmeove the containers and volumes

   ```bash
   docker-compose down -v
   ```

   :exclamation: This will remove the `openfire-app` and `openfire-db` containers and the volumes (but not the directories on your host :wink: ) We can safely do so because we only have host mounted volumes. Leave out the `-v` flag if you added any named voumes inside your modified `docker-compose.yml` file.

   Or you could do it more verbose:

   ```bash
   docker-compose stop
   docker-compose rm
   docker volume ls
   docker volume rm <openfire-data-volume-id>
   docker volume rm <openfire-logs-volume-id>
   docker volume rm <openfire-certs-volume-id>
   docker volume rm <openfire-db-volume-id>
   ```

2. Bump the version number for Openfire in the [Dockerfile](openfire/app/Dockerfile) to the new release's version e.g.
   ```Dockerfile
   ENV OPENFIRE_VERSION=4.2.3 \
   ```

3. Bring the containers back up:
   
   ```bash
   docker-compose up -d
   ```
   Because we have our docker-compose setup to build the image from the `./openfire/app/` directory it will setup a new openfire installation and pull all our persistent data back in from the host folders as volumes.

## Shell Access

For debugging and maintenance purposes you may want access the containers shell. If you are using Docker version `1.3.0` or higher you can access a running containers shell by starting `bash` using `docker exec`:

```bash
docker exec -it openfire-app bash
```

# References

  * http://www.igniterealtime.org/projects/openfire/
  * https://library.linode.com/communications/xmpp/openfire/ubuntu-12.04-precise-pangolin
