# docker-dive

> Something about docker, does not everything but something everyone should probably read

_Warning: This is a work in progress_
## Table of Contents

- [Containers 101](#containers-101)
- [Docker Architecture](#docker-architecture)
  - [How Container Runs](#how-container-runs)
  - [Experimental Features](#experimental-features)
  - [Resource Limitation](#resource-limitation)
  - [Using GPUs](#using-gpus)

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
- you need to install nvidia drivers & also nvidia-container-runtime
- check if you have nvidia: `lspci | grep -i nvidia
- `docker run -it --rm --gpus all ubuntu nvidia-smi`
- `docker run -it --rm --gpus device=GPU-3a23c669-1f69-c64e-cf85-44e9b07e7a2a ubuntu nvidia-smi`
- `docker run -it --rm --gpus device=0,2 ubuntu nvidia-smi`
- `docker run --gpus 'all,capabilities=utility' --rm ubuntu nvidia-smi`
- https://github.com/NVIDIA/nvidia-docker/wiki/CUDA

### GPU Run

```shell
docker run --gpus all -it --rm tensorflow/tensorflow:latest-gpu \
   python -c "import tensorflow as tf; print(tf.reduce_sum(tf.random.normal([1000, 1000])))"
```

## Docker Data Management

## Docker Network

### Port Mappings

## Security

## Deployment Tips

## Docker Testing/Linting

## Serverless functions with Containers

## Buildkit

<!-- links section -->
[aquasec-container-history]: https://blog.aquasec.com/a-brief-history-of-containers-from-1970s-chroot-to-docker-2016
[oci-home]: https://opencontainers.org/
[runc-gh]: https://github.com/opencontainers/runc
[docker-https]: https://docs.docker.com/engine/security/https/
[docker-cli-gh-experimental]: https://github.com/docker/cli/blob/master/experimental/README.md
