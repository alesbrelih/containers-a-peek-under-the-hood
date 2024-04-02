# Rootless containers

Docker runs a root daemon because it needs to be root to create namespaces.
But the is another option.

1. Try to create a namespace as nonroot.

```bash
root@k8s:/# useradd ales
root@k8s:/# su ales
$ unshare --uts sh
unshare: unshare failed: Operation not permitted
```

2. But we can create a new UTS namespace inside a new user namespace.

```bash
$ unshare --user --uts sh
$ id
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
```

And now we see that we run as nobody.

If we wanted to run as root we can also do that.

```bash
$ unshare --map-root-user --user --uts sh
# id
uid=0(root) gid=0(root) groups=0(root)
```

Lets create a file inside `/tmp`.

```bash
# touch /tmp/myfile
# ls -al /tmp/ | grep myfile
-rw-rw-r--  1 root   root       0 Apr  2 17:27 myfile
```

File is owned by root. But if we exit the namespace and recheck.

```bash
root@k8s:/# ll /tmp/ | grep myfile
-rw-rw-r--  1 ales ales    0 Apr  2 17:27 myfile
```

We see that even though we were root we weren't root on host. This is done using usernamespaces and UID_MAP.
This is what rootless containers use under the hood.

Rootless isn't enabled by default for Docker, we need to opt in.
Podman on the other hand uses this by default.
