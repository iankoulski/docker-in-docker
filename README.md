docker-in-docker
================

## Table of Contents
* [Docker EE (Engine, UCP and DTR)](#docker-ee-engine-ucp-and-dtr)
* [Image Tags](#image-tags)
* [Build images (optional)](#build-images-optional)
* [Prerequisites](#prerequisites)
* [Single engine](#single-engine)
* [Swarm mode cluster](#swarm-mode-cluster)

## Docker EE (Engine, UCP and DTR)
Stop! Look at [scripts](./scripts) for tools to automatically create a Swarm mode cluster, a UCP cluster, or a DDC (both UCP & DTR) cluster. You can even start a preconfigured Jenkins to use for demos. If you want to manually stand up an engine or Swarm mode cluster, read on.

## Image Tags
  * `ce`, `ce-stable`, `latest` ([Dockerfile.ce](./Dockerfile.ce))
  * `ce-test` ([Dockerfile.ce-test](./Dockerfile.ce-test))
  * `ce-edge` ([Dockerfile.ce-edge](./Dockerfile.ce-edge))
  * `ee` ([Dockerfile.ee](./Dockerfile.ee))
  * `ee-17.06` ([Dockerfile.ee-17.06](./Dockerfile.ee-17.03))
  * `ee-17.03` ([Dockerfile.ee-17.03](./Dockerfile.ee-17.03))
  * `ee-test` ([Dockerfile.ee-test](./Dockerfile.ee-test))
  * `cs-1.12` [Dockerfile.cs-1.12](./Dockerfile.cs-1.12))
  * `cs-1.10` [Dockerfile.cs-1.10](./Dockerfile.cs-1.10))
  * `cs-1.9` [Dockerfile.cs-1.9](./Dockerfile.cs-1.9))
  * `haproxy` ([Dockerfile.haproxy](./scripts/haproxy/Dockerfile.haproxy)) - used by  `dind_ddc`
  * `mirror` ([Dockerfile.mirror](./scripts/mirror/Dockerfile.mirror)) - used by  `dind_ddc`

## Build images (optional)
<details><summary>Expand for more details</summary><p>

The images are published to Docker Hub so you do not need to build them unless you want to,

*Note*: your `<DOCKER-EE-URL>` value can be found from https://store.docker.com/?overlay=subscriptions

* Docker CE (stable)
  ```
  docker build \
    -t mbentley/docker-in-docker:ce-17.03 \
    -t mbentley/docker-in-docker:ce \
    -f Dockerfile.ce .
  ```
* Docker CE (test)
  ```
  docker build \
    -t mbentley/docker-in-docker:ce-test \
    -f Dockerfile.ce-test .
  ```
* Docker CE (edge)
  ```
  docker build \
    -t mbentley/docker-in-docker:ce-edge \
    -f Dockerfile.ce-edge .
  ```
* Docker EE (stable)
  ```
  docker build \
    --build-arg DOCKER_EE_URL="<DOCKER-EE-URL>" \
    -t mbentley/docker-in-docker:ee \
    -f Dockerfile.ee .

  docker push mbentley/docker-in-docker:ee
  ```
* Docker EE (stable-17.03)
  ```
  docker build \
    --build-arg DOCKER_EE_URL="<DOCKER-EE-URL>" \
    -t mbentley/docker-in-docker:ee-17.03 \
    -f Dockerfile.ee-17.03 .

  docker push mbentley/docker-in-docker:ee-17.03
  ```
* Docker EE (stable-17.06)
  ```
  docker build \
    --build-arg DOCKER_EE_URL="<DOCKER-EE-URL>" \
    -t mbentley/docker-in-docker:ee-17.06 \
    -f Dockerfile.ee-17.06 .

  docker push mbentley/docker-in-docker:ee-17.06
  ```
* Docker EE (test)
  ```
  docker build \
    --build-arg DOCKER_EE_URL="<DOCKER-EE-URL>" \
    -t mbentley/docker-in-docker:ee-test \
    -f Dockerfile.ee-test .

  docker push mbentley/docker-in-docker:ee-test
  ```

* Docker CS Engine 1.12
  ```
  docker build \
    -t mbentley/docker-in-docker:cs-1.12 \
    -f Dockerfile.cs-1.12 .

  docker push mbentley/docker-in-docker:cs-1.12
  ```

* Docker CS Engine 1.10
  ```
  docker build \
    -t mbentley/docker-in-docker:cs-1.10 \
    -f Dockerfile.cs-1.10 .

  docker push mbentley/docker-in-docker:cs-1.10
  ```

* Docker CS Engine 1.9
  ```
  docker build \
    -t mbentley/docker-in-docker:cs-1.9 \
    -f Dockerfile.cs-1.9 .

  docker push mbentley/docker-in-docker:cs-1.9
  ```
</p></details>

## Prerequisites
  * Docker for Mac installed
  * Must have the following ports available on your host:
    * `1000` - TCP connection to a single Docker engine (or whatever you specify)
    * `1001`, `1002`, `1003` - TCP connection to Docker engines for Swarm mode (or whatever you specify)

## Single engine

1. Start engine
    ```
    docker run -d \
      --name docker \
      --privileged \
      -p 127.0.0.1:1000:2375 \
      -v /lib/modules:/lib/modules:ro \
      -v docker:/var/lib/docker \
      -v docker-etc:/etc/docker \
      --tmpfs /run \
      mbentley/docker-in-docker \
      dockerd -s overlay2 -H unix:///var/run/docker.sock -H tcp://0.0.0.0:2375
    ```

2. Communicate with that engine
    ```
    docker -H tcp://localhost:1000 info
    ```

3. Check version
    ```
    docker -H tcp://localhost:1000 version
    ```

4. Destroy the Engine
    ```
    docker kill docker
    docker rm docker
    docker volume rm docker
    ```

## Swarm mode cluster
1. Create 3 engines
    ```
    for ENGINE_NUM in {1..3}
    do
      docker run -d \
        --name docker${ENGINE_NUM} \
        --privileged \
        -p 127.0.0.1:100${ENGINE_NUM}:2375 \
        -v /lib/modules:/lib/modules:ro \
        -v docker${ENGINE_NUM}:/var/lib/docker \
        -v docker${ENGINE_NUM}-etc:/etc/docker \
        --tmpfs /run \
        mbentley/docker-in-docker \
        dockerd -s overlay2 -H unix:///var/run/docker.sock -H tcp://0.0.0.0:2375
    done
    ```

2. Create a new Swarm
    ```
    docker -H tcp://localhost:1001 swarm init
    ```

3. Get the worker join token and command
    ```
    TOKEN=$(docker -H tcp://localhost:1001 swarm join-token worker -q)
    JOIN_COMMAND="swarm join --token ${TOKEN} $(docker container inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' docker1):2377"
    ```

4. Join engine 2
    ```
    docker -H tcp://localhost:1002 ${JOIN_COMMAND}
    ```

5. Join engine 3
    ```
    docker -H tcp://localhost:1003 ${JOIN_COMMAND}
    ```

6. Check status
    ```
    docker -H tcp://localhost:1001 node ls
    ```

7. Destroy Swarm cluster
    ```
    docker kill docker1 docker2 docker3
    docker rm docker1 docker2 docker3
    docker volume rm docker1 docker2 docker3
    ```
