# Container is just a process.

1. Spawn a new container in detached mode.

```bash
root@k8s:~# docker run --rm -d alpine sleep 100
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
4abcf2066143: Pull complete
Digest: sha256:c5b1261d6d3e43071626931fc004f70149baeba2c8ec672bd4f27761f8e1ad6b
Status: Downloaded newer image for alpine:latest
e08b29215bd15bf1a5debdebd6881f4dd9b1e51692609b85a38f586954c069bd
```

`--rm` flag automatically removes container when it's done.

2. Search for container under processes by searching for a process running sleep command.

```bash
root@k8s:~# ps -fC sleep
UID          PID    PPID  C STIME TTY          TIME CMD
root       38061   38041  0 16:38 ?        00:00:00 sleep 100
```
