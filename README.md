# docker-dive
Something about docker, does not everything but something everyone should probably read

## Table of Contents

- [Containers 101](#containers-101)
- [Docker Architecture](#docker-architecture)
  - [How Container Runs](#how-container-runs)

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

## Resource Limitation

## Using GPUs

## Docker Data Management

### Port Mappings

## Docker Network

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
