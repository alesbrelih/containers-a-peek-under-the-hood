# Isolating process IDs and using chroot.

Isolating process IDs is important because process/container shouldn't see processes on the host because that is another attack vector.

1. Create a new process ID namespace

```bash
unshare --pid --fork sh
```

2. List processes

```sh
ps -ef
```

Processes are still shown what happened?

3. Check process ID of spawned shell.

```sh
# echo $$
1
```

So we've created a new process namespace but we can still see processes on the host. Not good.

4. Exit back to host.

```sh
# exit
```

5. Turns out that ps reads processes from `/proc` pseudo-filesystem (info about kernel). So we need a new `/` (root) and `/proc`.

```bash
mkdir /tmp/newroot
```

6. Download alpine mini-root filesystem.

```bash
wget https://dl-cdn.alpinelinux.org/alpine/v3.19/releases/x86_64/alpine-minirootfs-3.19.1-x86_64.tar.gz -O - | tar -xz -C /tmp/newroot
```

7. Create a new PID namespace with new root.

```sh
unshare --pid --fork chroot /tmp/newroot sh
```

8. Mount `/proc` pseudo-filesystem.

```sh
mount -t proc proc /proc
```

9. List processes.

```sh
/ # ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 sh
    3 root      0:00 ps -ef
```

Tada!

10. Turns out we can be smarter and we don't need a new root we just need a separate mount namespace where we can mount proc.

On host:

```bash
unshare --pid --fork --mount-proc sh
```

```sh
# hostname newhost
```

5. Checking hostname

```sh
# hostname
newhost
```

6. Exiting the namespace

```sh
# exit
```

7. Checking hostname

```sh
# hostname
k8s
```

Which means changes inside namespaces aren't reflected to the host.
