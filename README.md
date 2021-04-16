# docker-dive

> Something about docker, does not everything but something everyone should probably read

_Warning: This is a work in progress thus some sections may not be fully accurate until thhis wip warning is removed_
## Table of Contents

- [Containers 101](#containers-101)
- [Docker Architecture](#docker-architecture)
  - [How Container Runs](#how-container-runs)
- [Experimental Features](#experimental-features)
- [Resource Limitation](#resource-limitation)
- [Using GPUs](#using-gpus)
  - [GPU Run](#gpu-run)
- [Docker Data Management](#docker-data-management)
  - [Storage Drivers](#storage-drivers)
  - [Data Volumes](#data-volumes)
  - [Bind Mounts](#bind-mounts)
- [Docker Network](#docker-network)
  - [Port Mappings](#port-mappings)
- [Security](#security)
- [Deployment Tips](#deployment-tips)
- [Docker Testing/Linting](#docker-testinglinting)
- [Serverless functions with Containers](#serverless-functions-with-containers)
- [Buildkit](#buildkit)

## Containers 101

- idea behind containers is nothing new. In fact, chroot jail (roughly 1979), process containers with cgroups (roughly 2006), lxc (roughly 2008) and others were critical in leading us to the path that we're walking right now. [This post][aquasec-container-history] provides more detailed view of container history.
- Industry moving towards [Open Container Initiative(OCI)][oci-home] to create open standards around container formats and runtimes.
- OCI would help us have interoperable image format rather than various competing standards such as docker, appc, lxd that may be incompatible for each other.
- OCI would help us have interoperable runtime rather than various competing container engines (docker, cri-o, railcar, rkt, lxc).
- [runc][runc-gh] provides reference runtime implementation for OCI specification and it comes as a CLI tool for spawning and running containers according to the OCI specification.

## Docker Architecture

- Docker models after client-server architecture.
- Docker daemon running on a particular host is accessible through one of `unix` socket, `tcp` socket or via `fd`.
- To access docker daemon remotely, you need to enable `tcp` socket.
- The default setup provides un-encrypted and un-authenticated direct access to the Docker daemon so it must always be secured either using the built in [HTTPS encrypted socket][docker-https], or by putting a secure web proxy in front of it

### How container Runs

When you type a particular command in your shell in Linux,
the shell makes a request to the kernel to create a normal
process using a flavor of the `exec` syscall family.

Lets validate that with an example:

```shell
$ strace ls
execve("/bin/ls", ["ls"],...
```

A container process is however way more different and special
because when you ask docker daemon to start a docker container,
it makes a request to kernel to create a process using different
flavor of syscall called `clone`. The `clone` syscall is special
because it can create a process with separate namespace (man 7 namespaces) i.e.
the process could have its own virtual mount points, process ids,
user ids, network interfaces, etc.

Lets validate that with an example:

```shell
$ strace docker run --rm -it nginx
execve("/usr/bin/docker", ["docker", "run", "--rm", "-it", "nginx"], 0x7ffdf133d9e0 /* 108 vars */) = 0
brk(NULL)                               = 0x557742d57000
......
......
clone(child_stack=0x7fd5d50a4fb0, flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, parent_tid=[624898], tls=0x7fd5d50a5700, child_tidptr=0x7fd5d50a59d0) = 624898
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigprocmask(SIG_SETMASK, ~[RTMIN RT_1], [], 8) = 0
mmap(NULL, 8392704, PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0) = 0x7fd5d40a4000
mprotect(0x7fd5d40a5000, 8388608, PROT_READ|PROT_WRITE) = 0
clone(child_stack=0x7fd5d48a3fb0, flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, parent_tid=[624899], tls=0x7fd5d48a4700, child_tidptr=0x7fd5d48a49d0) = 624899
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
rt_sigprocmask(SIG_SETMASK, ~[RTMIN RT_1], [], 8) = 0
mmap(NULL, 8392704, PROT_NONE, MAP_PRIVATE|MAP_ANONYMOUS|MAP_STACK, -1, 0) = 0x7fd5ceffe000
mprotect(0x7fd5cefff000, 8388608, PROT_READ|PROT_WRITE) = 0
clone(child_stack=0x7fd5cf7fdfb0, flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|CLONE_CHILD_CLEARTID, parent_tid=[624901], tls=0x7fd5cf7fe700, child_tidptr=0x7fd5cf7fe9d0) = 624901
rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
futex(0xc00009ebc8, FUTEX_WAKE_PRIVATE, 1) = 1
futex(0xc0000cc148, FUTEX_WAKE_PRIVATE, 1) = 1
```

## Experimental Features

Docker ships with various experimental features and before they make it as stable features (or happen to be kicked out), you can use those features.

To enable experimental features on docker daemon, you can edit the `daemon.json` (you can create one if it doesn't exist) which typically sits in `/etc/docker/daemon.json` on Linux hosts.

```json
{
  "features": { "buildkit": true },
  "experimental": true
}
```

To enable experimental features on Docker CLI client, you can edit the `$HOME/.docker/config.json` on Linux hosts:

```json
{
  "experimental": "enabled"
}
```

Docker cli project maintains the experimental feature lists [HERE][docker-cli-gh-experimental] but I have not found the same for Docker daemon. However, most dockerd documentations should mention whenever a feature is experimental.

## Resource Limitation

By default, a container has no resource constraints and can use as much of a given resource as the host’s kernel scheduler allows. Docker provides ways to control how much memory, or CPU a container can use, setting runtime configuration flags of the docker run command. This section provides details on when you should set such limits and the possible implications of setting them.

Few examples follow:

```shell
docker container run -d --name test-nginx --cpu-shares 512 --memory 128M -p 8080:80 nginx
docker container update --cpu-shares 512 --memory 128M --memory-swap 256M test-nginx
docker -d --storage-opt dm.basesize=5G --cpuset=0,1 -c 2 -m 128m
--cpus="1.5" === --cpu-period="100000" --cpu-quota="150000"
--cpuset-cpus 0-3 or 1,3

docker info to see details on what's supported & what's not

--oom-kill-disable
--oom-score-adj (-1000 to 1000; lower means less chance to be killed on oom)
```

## Using GPUs

- supports nvidia currenty
- you need to install nvidia drivers & also [nvidia-container-runtime][nvidia-runc-gh]
- check if you have nvidia: `lspci | grep -i nvidia`
- `docker run -it --rm --gpus all ubuntu nvidia-smi`
- `docker run -it --rm --gpus device=GPU-3a23c669-1f69-c64e-cf85-44e9b07e7a2a ubuntu nvidia-smi`
- `docker run -it --rm --gpus device=0,2 ubuntu nvidia-smi`
- `docker run --gpus 'all,capabilities=utility' --rm ubuntu nvidia-smi`
- [CUDA images][cuda-gh-wiki] are available to download from nvidia's [hub repo][nvidia-hub]

### GPU Run

```shell
docker run --gpus all -it --rm tensorflow/tensorflow:latest-gpu \
   python -c "import tensorflow as tf; print(tf.reduce_sum(tf.random.normal([1000, 1000])))"
```

## Docker Data Management

- volumes - non-docker process should not modify; best way to persist data in docker
- bind mounts - from anywhere in host system
- tmpfs

### Storage Drivers

- overlay2 and overlay based on unionfs
- brtfs
- devicemapper

### Data Volumes

Volumes are the preferred mechanism for storing data accessed (read/write) by a container. Volumes are managed by docker, and can be shared with multiple containers.

```
# create a volume named nginx-vol
$ docker volume create nginx-vol

# map the volume via --mount arg on docker run
$ docker run -d \
  --name=nginxtest \
  --mount source=nginx-vol,destination=/usr/share/nginx/html \
  nginx:latest

# or map the volume via -v arg on docker run
$ docker run -d \
  --name=nginxtest \
  -v nginx-vol:/usr/share/nginx/html \
  nginx:latest

# delete the volume
$ docker volume rm nginx-vol
```

Note that lifecycles of volume and the container are independent thus docker does not automatically delete the volume when a container is deleted and vice versa. You can use `docker rm -v` if you want to delete the volume when you are deleting the container.


### Bind Mounts

You can use bind mounts to mount host directory and while this is really useful for development and testing, it should ideally only be limited for the dev/test purposes. In general, use data volumes to store data accessed by a container. The volumes are managed by docker itself and you can share the volumes across multiple containers in different mix of access modes.

```
# bind the local path using --mount arg
$ docker run -d -P \
	--name web \
	--mount type=bind,source=/projects/app,target=/opt/app \
	your/image \
  some-daemon-process

# or bind the local path using -v arg
$ docker run -d -P \
	--name web \
	-v /projects/app:/opt/app \
	--mount type=bind,source=/projects/app,target=/opt/app \
	your/image \
  some-daemon-process
```

## Docker Network

Like rest of the other namespaces, docker makes use of network namespace
to achieve isolation of the networking resources in a system.
This allows docker containers to have separate interfaces,
routes and IPs making it totally flexible to manage networks in a docker container.

Like many of the other subsystems, Docker's networking subsystem
is very pluggable with network drivers. The default drivers that exist by default are:

- bridge
- host (linux-only)
- overlay
- macvlan
- none

### Port Mappings

```shell
$ docker run -d -p 127.0.0.1:3000:3000 my-image

$ docker run -d -p 127.0.0.1::3000 my-image

$ docker run -d -p 127.0.0.1:3000:3000/udp my-image
```

## Security

## Deployment Tips

- Never use latest tag as it is harder to track which version of the image is running and more difficult to roll back properly. [Kubernetes][k8-label-tag] recommends the same thing.
- For consistent and exactly reproducible builds, always use specific fully-specified base images. For example, prefer `node:15.2.1-alpine3.10` instead of `node:15-alpine3.10`, `node:current-alpine3.10` and `node:alpine3.10`. Although these tags may point at the `15.2.1-alpine3.10` tag at particular point of time, that will change over time as things change with nodejs releases.
- Make sure you are only providing just enough permissions for your app to run in your container. For example, run the app as non-root user if you don't need root to run the app.
- Make sure you have appropriate signal handlers (SIGTERM, SIGINT & then SIGKILL) in your app. This is useful for cleaning up but still handling in-flight user requests, for example. Also, take a look at project like [tini][tini-gh] for having proper `init` system for your containers. If you are using Docker 1.13 or greater including the Docker CE, Tini is included in Docker itself and you can enable tini by passing `--init` flag to `docker run` command.
- We already talked about this on security section above but its critical to have vulnerability scans on your docker images and also on your containers.

## Docker Testing/Linting

## Serverless functions with Containers

## Buildkit

### Why buildkit

- concurrent
- cache-efficient
- dockerfile-agnostic
- `DOCKER_BUILDKIT=1 docker build` or `docker buildx`
- Automatic garbage collection
- Efficient instruction caching
- Multiple output formats
- Distributable workers

### How to enable
- `{ "features": { "buildkit": true } }` in `/etc/docker/daemon.json`
- The new syntax features in Dockerfile are available if you override the default frontend. To override the default frontend, set the first line of the Dockerfile as a comment with a specific frontend image: `# syntax = <frontend image>, e.g. # syntax = docker/dockerfile:1.0-experimental`

### Sample dockerfile with secrets:
- A secret file is automatically mounted only to a separate tmpfs filesystem to make sure that it does not leak to the final image nor to the next command and that it does not remain in the local build cache.

```docker
# syntax = docker/dockerfile:1.0-experimental
FROM alpine

# shows secret from default secret location:
RUN --mount=type=secret,id=mysecret cat /run/secrets/mysecret

# shows secret from custom secret location:
RUN --mount=type=secret,id=mysecret,dst=/foobar cat /foobar

# docker build --no-cache --progress=plain --secret id=mysecret,src=mysecret.txt .
```

### Sample dockerfile with ssh

- Probably the most popular use case for accessing private data in builds is for getting access to private repositories through SSH
- use the `--ssh` flag to forward your existing SSH agent connection or a key to the builder. Instead of transferring the key data, docker will just notify the builder that such capability is available. Now when the builder needs access to a remote server through SSH, it will dial back to the client and ask it to sign the specific request needed for this connection. The key itself never leaves the client, and as soon as the command that requested the access has completed there is no information on the builder side to reestablish that remote connection later.
- The docker build has a --ssh option to allow the Docker Engine to forward SSH agent connections
- Only the commands in the Dockerfile that have explicitly requested the SSH access by defining type=ssh mount have access to SSH agent connections. The other commands have no knowledge of any SSH agent being available.
- To request SSH access for a RUN command in the Dockerfile, define a mount with type ssh. This will set up the `SSH_AUTH_SOCK` environment variable to make programs relying on SSH automatically use that socket.

```docker
# syntax=docker/dockerfile:experimental
FROM alpine

# Install ssh client and git
RUN apk add --no-cache openssh-client git

# Download public key for github.com
RUN mkdir -p -m 0600 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts

# Clone private repository
RUN --mount=type=ssh git clone git@github.com:myorg/myproject.git myproject

# docker build --ssh default .
```

```
RUN --mount=type=ssh,id=projecta git clone projecta
RUN --mount=type=ssh,id=projectb git clone projectb

docker build --ssh projecta=./projecta.pem --ssh projectb=./projectb.pem

# Note that even when actual keys are specified here, still only the agent connection is shared with the builder and not the actual content of these private keys
```

### Multi-platform images

<!-- links section -->
[aquasec-container-history]: https://blog.aquasec.com/a-brief-history-of-containers-from-1970s-chroot-to-docker-2016
[oci-home]: https://opencontainers.org/
[runc-gh]: https://github.com/opencontainers/runc
[docker-https]: https://docs.docker.com/engine/security/https/
[docker-cli-gh-experimental]: https://github.com/docker/cli/blob/master/experimental/README.md
[nvidia-runc-gh]: https://github.com/NVIDIA/nvidia-container-runtime
[cuda-gh-wiki]: https://github.com/NVIDIA/nvidia-docker/wiki/CUDA
[nvidia-hub]: https://hub.docker.com/r/nvidia/cuda
[k8-label-tag]: https://kubernetes.io/docs/concepts/configuration/overview/#using-labels
[tini-gh]: https://github.com/krallin/tini
