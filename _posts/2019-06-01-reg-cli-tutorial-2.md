---
title: Docker Registry CLI Tutorial - More Use
categories: [tech]
tags: [docker, shell, studio, studio-reg-cli]
toc: true
toc_sticky: true
---

The open source project [Docker Registry CLI](https://github.com/morningspace/docker-registry-cli) is released by MorningSpace Studio! This post tells you more advanced use.

![](/assets/images/studio/docker_registry.png)

## Copy more images

Let’s copy more images to the registry. `reg-cli cp` can copy multiple images at a time. Just append the images after the command and separate them by space.

```shell
bash-4.4# reg-cli cp morningspace/docker-registry-cli morningspace/elastic-shell mr.io
```

This will copy both `docker-registry-cli` and `elastic-shell` images to `mr.io`. Apply the same pattern to `reg-cli ls`, we can list tags for multiple images.

```shell
bash-4.4# reg-cli ls mr.io/docker-registry-cli mr.io/elastic-shell
```

Of course you can use `-ld` and `-lm` options to list digests and manifests for these images.

## Copy in batch using .list file

`reg-cli cp` can also support to read from configuration files. We can use the registry name as the file name and `.list` as the extension name, add the name of each image that you want to copy to the file, one image per line. Then run `reg-cli cp`, and specify the registry name. It will find the corresponding `.list` file in current folder by using the registry name and read the images from the file one by one, then copy them to the target registry.

There are some examples distributed with the cli on the [Git repository](https://github.com/morningspace/docker-registry-cli) at the folder `/samples/registries`. Let’s take `morningspace.list` as example, it includes some images where the names are all started with `morningspace`. These images are all from [`morningspace`](https://hub.docker.com/u/morningspace).

```
morningspace/lab-web
morningspace/lab-lb
morningspace/lab-git-remote
morningspace/lab-git-local
morningspace/elastic-shell
morningspace/docker-registry-cli
```

Move to that folder, and run `reg-cli cp`. That will copy all these images from Docker Hub to `mr.io`.

Moreover, `.list` files can be nested. For example, let’s check `all.list`. It has two entries, where each one references the corresponding `.list` file stored in the same folder.

```
k8s.gcr.io
morningspace
```

Let’s run the reg-cli cp for all.

```shell
bash-4.4# reg-cli cp all mr.io
```

When finished, let’s run `reg-cli ls` to list the catalog of `mr.io`.

```shell
bash-4.4# reg-cli ls mr.io
mr.io
---------------------------------------
coredns
docker-registry-cli
elastic-shell
etcd
hyperkube
kubernetes-dashboard-amd64
lab-git-local
lab-git-remote
lab-lb
lab-web
pause
```

You will see images that are configured in the two `.list` files are all copied to `mr.io`.

Again, we can use `reg-cli ls` with multiple images to list tags for these images, use `-ld` to list digests, and use `-lm` to list manifests.

We can even apply these options to the whole registry, to list all images, their tags, digests and manifests. For example, let’s use `-lt` to list tags for all images, and append `less` as it may be too tedious to read the whole bunch of these contents.

```shell
bash-4.4# reg-cli ls mr.io -lt
mr.io/coredns
---------------------------------------
1.2.6

mr.io/docker-registry-cli
---------------------------------------
latest

mr.io/elastic-shell
---------------------------------------
latest
v0.3
...
```

As you can see, we only used two commands with just a few options. But they can be combined in many different ways to do a lot of things.

## Remove images or tags

Now, let’s remove some images from the registry. We can run `reg-cli rm` to remove just one tag for an image.

```shell
bash-4.4# reg-cli rm mr.io/hyperkube:v1.13.0
reg-cli:(INFO) going to remove hyperkube:v1.13.0
Are you sure? [y/N] y
reg-cli:(INFO) digest: sha256:8ef688eb5bb739e7f22502e15b78eb6bd684d2957dfda7c64638cec0e9eacb3d
reg-cli:(INFO) ✔ hyperkube:v1.13.0 removed
```

The tag is removed after we input y to confirm.

Also, we can remove multiple tags for the same image in one line.

```shell
bash-4.4# reg-cli rm mr.io/hyperkube:v1.13.0,v1.12.1
```

If there’s no tag appended, it will remove all tags for that image.

```shell
bash-4.4# reg-cli rm mr.io/hyperkube
```

And, if you want to enforce the removal, you can use `-f` option.

```shell
bash-4.4# reg-cli rm mr.io/hyperkube:v1.13.0,v1.12.1 -f
```

(The end)

---
*Have fun!*

*MorningSpace Studio*