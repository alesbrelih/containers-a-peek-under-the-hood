# Root inside container is a root on host

1. Spawn a new container in interactive mode.

```bash
root@k8s:/tmp/archive# docker run --rm -it alpine sh
/ # id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
```

2. Run long running process inside container

```bash
sleep 100
```

3. Search for container under host processes (inside another terminal) by searching for a process running sleep command.

```bash
root@k8s:~# ps -fC sleep
UID          PID    PPID  C STIME TTY          TIME CMD
root       38061   38041  0 16:38 ?        00:00:00 sleep 100
```

So basically ROOT inside the container is ROOT on the host.

4. Why is this bad?

Let's say I'm inside docker group but I'm not root. I can just spawn a new container with root mounted inside container and I can change any file.

```bash
root@k8s:/# docker run --rm -it -v /:/tmp/root alpine sh
/ # touch /tmp/root/root/hiiamroot
/ # exit
root@k8s:/# ll /root/ | grep hiiam
-rw-r--r--  1 root root    0 Apr  2 17:21 hiiamroot
```
