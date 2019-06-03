---
title: Docker Registry CLI Tutorial - Basic Use
categories: [tech]
tags: [docker, shell, studio, studio_reg_cli]
toc: true
toc_sticky: true
---

The open source project [Docker Registry CLI](https://github.com/morningspace/docker-registry-cli) is released by MorningSpace Studio! This post tells you the basic use.

![](/assets/images/studio/docker_registry.png)

## What is it?

If you use Docker, you may know that [Docker Registry](https://docs.docker.com/registry/) is a service that is used to store Docker images. There are some popular public Docker registries, such as [Docker Hub](https://hub.docker.com/), [Quay](https://quay.io/), [Google Container Registry](https://cloud.google.com/container-registry/)，[AWS Container Registry](https://aws.amazon.com/ecr/), etc. People can also build their own private Docker registry.

[Docker Registry CLI](https://github.com/morningspace/docker-registry-cli), is an open source command line utility written in Bash Shell to manage Docker registry.

## Why different?

There are some [alternatives](https://github.com/search?q=docker+registry+cli&type=Repositories) that can be found on GitHub. Most of them are written in Go or Python. Here are some highlights to help you understand why Docker Registry CLI is different from others.

* *Copy in batch*: To copy images in batch is one interesting feature provided by the cli. It can be used to copy multiple images from different sources including both private and public registries to the target registry so that you can setup your own private registry easily.
* *Based on DIND*: The cli provides a Docker image based on [DIND(Docker-in-Docker)](https://github.com/jpetazzo/dind). By using DIND, not only the cli can be run inside container, it can even run docker commands in container without polluting the local registry cache on your host machine.
* *Less is more*: With the combination of a very little set of commands and options, it provides quite a few features to manipulate the registry. See "[How to Use](#how-to-use)" for details.
* *Easy to use*: The design rationale behind the cli is to reference existing linux command syntax as much as possible. So, it's fairly easy to learn and use if you are familiar with some ordinary linux commands such as `ls`, `rm`, and `cp`.

Next, let’s start to look at how to use Docker Registry CLI to setup a private registry quickly. We will see how these highlights are reflected by going through the demo.

## How to run?

Docker Registry CLI can be run either inside or outside Docker container. It's much easier to run inside container because it has all dependencies installed and a soft link created for the shell script so that you can run the cli from anywhere in container.

The Docker Registry CLI image has been built and published to [Docker Hub](https://cloud.docker.com/u/morningspace/repository/docker/morningspace/docker-registry-cli). So, you can pull it from there and use it.

Another way to run the cli is to clone the [Git repository](https://github.com/morningspace/docker-registry-cli) directly to your local. Then use Docker Compose to run it. The `docker-compose.yml` file includes the cli with a Docker daemon wrapped as a container, and a simple private registry for testing purpose, where the hostname is `mr.io`. In this post, we will use this approach to run the demo.

So, let’s firstly start both the Docker daemon and the registry:

```shell
$ docker-compose up -d
Creating network "net-registry" with driver "bridge"
Creating docker-registry-cli-v0_reg-cli_1_d4de93c59ec0 ... done
Creating mr.io                                         ... done
```

Then, launch another container to connect to the Docker daemon:

```shell
$ docker-compose exec reg-cli bash
```

## Copy your first image

In order to copy images to the registry, we need `reg-cli cp`. By running the command, it can copy images in batch from source registries including both public and private registries to the target registry.

Let’s copy the Docker Registry CLI image itself from Docker Hub to our `mr.io` registry.

```shell
bash-4.4# reg-cli cp morningspace/docker-registry-cli mr.io
```

Then, let’s use `reg-cli ls` to list the catalog of `mr.io`.

```shell
bash-4.4# reg-cli ls mr.io
mr.io
---------------------------------------
docker-registry-cli
```

Congratulations! You’ve successfully copied your first image to `mr.io`. Append the image name after `reg-cli ls`, we can list all tags of this image.

```shell
bash-4.4# reg-cli ls mr.io/docker-registry-cli
mr.io/docker-registry-cli
---------------------------------------
latest
```

Right now, we only have the `latest` tag. We can also use `-ld` option to list digests for the image, and `-lm` option to list manifests for the image.

```shell
bash-4.4# reg-cli ls mr.io/docker-registry-cli -ld
mr.io/docker-registry-cli
---------------------------------------
sha256:d1b110ad26922d45fce7baccd572ba35eef0412a0af8354177ae742d99718992 latest
```

```shell
bash-4.4# reg-cli ls mr.io/docker-registry-cli -lm
mr.io/docker-registry-cli
---------------------------------------
{
   "schemaVersion": 1,
   "name": "docker-registry-cli",
   "tag": "latest",
   "architecture": "amd64",
   "fsLayers": [
     ...
   ],
   "history": [
     ...
   ],
   "signatures": [
     ...
   ]
}
```

As we mentioned earlier, Docker Registry CLI image is based on [DIND(Docker-in-Docker)](https://github.com/jpetazzo/dind), so we can run docker commands in the container. Actually, what `reg-cli cp` does is to run docker commands to finish the work. Let’s run `docker images`:

```shell
bash-4.4# docker images
REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
morningspace/docker-registry-cli   latest              84f9cabf011f        10 hours ago        189MB
mr.io/docker-registry-cli          latest              84f9cabf011f        10 hours ago        189MB
```

As you see, this is the image that we pulled from Docker Hub just now and the tagged one that’s going to be pushed to `mr.io`. All these are invisible to the host machine. If you exist the container after get the work done, nothing will be left.

(To be continued)

---
*Have fun!*

*MorningSpace Studio*