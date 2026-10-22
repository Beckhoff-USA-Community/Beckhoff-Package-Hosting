# Docker images for TwinCAT containers

TwinCAT for Linux can run TwinCAT functions as Docker containers on Beckhoff RT Linux. Beckhoff describes it in the Beckhoff RT Linux documentation, under [building the Docker image](https://infosys.beckhoff.com/content/1033/beckhoff_rt_linux/18744671115.html). A container starts from an image, and images come from a registry: Docker Hub, unless the image name says otherwise.

Nexus can be that registry too. A docker (hosted) repository keeps the images for your controllers on the same server as your packages. This page sketches the idea with a small example: take Debian's slim image, add one thing, push it to Nexus, and pull it on a controller.

## On Nexus

* Under *Administration*, *Security*, *Realms*, add the *Docker Bearer Token Realm* to the active realms. The Docker client logs in through it.
* *Repositories*, *Create repository*, **docker (hosted)**, named `docker-hosted`. Pick **path-based routing** (Nexus 3.83 and later): the repository name then becomes part of the image name, and Docker goes through the same reverse proxy as everything else. Turn on **Allow anonymous docker pull** to match the anonymous read of the feeds, and set the deployment policy to *Disable redeploy*, so a version tag never changes once pushed.

Sonatype's [Docker documentation](https://help.sonatype.com/en/docker-registry.html) has the details, including the port connectors older servers use instead.

## An example image

On any machine with Docker, pull Debian's slim image. Debian 13 is the release Beckhoff RT Linux is built on:

```sh
docker pull debian:trixie-slim
```

Add the Mosquitto command line clients, a handy tool to see what is on the MQTT broker. A file named `Dockerfile` in an empty folder:

```dockerfile
FROM debian:trixie-slim
RUN apt-get update \
 && apt-get install -y --no-install-recommends mosquitto-clients \
 && rm -rf /var/lib/apt/lists/*
```

Build it under the full name it will have on Nexus, log in with a Nexus user that may write to `docker-hosted`, and push it:

```sh
docker build -t nexus.example.com/docker-hosted/mqtt-tools:1.0.0 .
docker login nexus.example.com
docker push nexus.example.com/docker-hosted/mqtt-tools:1.0.0
```

The image name is the address: server, repository, image, tag. `docker login` takes the server name only.

On a controller, the same name pulls it and runs it:

```sh
sudo docker pull nexus.example.com/docker-hosted/mqtt-tools:1.0.0
sudo docker run --rm nexus.example.com/docker-hosted/mqtt-tools:1.0.0 mosquitto_sub -h mqtt.example.com -u <user> -P <password> -t 'MyNetwork/+/info' -v
```

The second line lists every device connected to the ADS-over-MQTT network `MyNetwork` from the [route file](../examples/files.md#a-route-file).

## What a controller needs

* **The full image name** in every `docker pull` and in the `image:` line of a Compose file. Without the server in front, Docker goes to Docker Hub.
* **A certificate Docker trusts.** One from a public authority needs nothing. One from an in-house authority goes to `/etc/docker/certs.d/nexus.example.com/ca.crt`.
* **A login, only if anonymous pulls are off:** `sudo docker login nexus.example.com` once per controller. Docker stores the password unencrypted in `/root/.docker/config.json`, so give the controllers a Nexus user that can only read.
* **An image for its processor.** An image is built for the architecture of the machine that builds it.

## TwinCAT containers

Beckhoff's [sample configuration](https://infosys.beckhoff.com/content/1033/beckhoff_rt_linux/19106559243.html) builds an image with the TwinCAT runtime, installed from Beckhoff's package server with a myBeckhoff login passed in as a build secret. Pushed to Nexus, that image reaches other controllers without the login and without a build on each one.

The sample also carries its ADS-over-MQTT configuration inside the image. A route file holds the broker's user and password, so an image with one inside is like a package with a [secret in it](../packages/packing.md#secrets-in-a-package): readable by everyone who can pull it. Keep such a registry on the company network.
