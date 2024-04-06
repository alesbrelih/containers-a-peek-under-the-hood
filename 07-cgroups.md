# Control Groups (cgroups)

To limit process resources we can use cgroups.

1. To see existing cgroups:

Inside folder:

```bash
root@ubuntu-s-1vcpu-2gb-fra1-01:~# ls -l /sys/fs/cgroup/
total 0
-r--r--r--  1 root root 0 Apr  5 12:50 cgroup.controllers
-rw-r--r--  1 root root 0 Apr  5 12:51 cgroup.max.depth
-rw-r--r--  1 root root 0 Apr  5 12:51 cgroup.max.descendants
-rw-r--r--  1 root root 0 Apr  5 12:50 cgroup.procs
-r--r--r--  1 root root 0 Apr  5 12:51 cgroup.stat
-rw-r--r--  1 root root 0 Apr  5 12:52 cgroup.subtree_control
-rw-r--r--  1 root root 0 Apr  5 12:51 cgroup.threads
-rw-r--r--  1 root root 0 Apr  5 12:51 cpu.pressure
-r--r--r--  1 root root 0 Apr  5 12:51 cpu.stat
-r--r--r--  1 root root 0 Apr  5 12:51 cpuset.cpus.effective
-r--r--r--  1 root root 0 Apr  5 12:51 cpuset.mems.effective
drwxr-xr-x  2 root root 0 Apr  5 12:51 dev-hugepages.mount
drwxr-xr-x  2 root root 0 Apr  5 12:51 dev-mqueue.mount
drwxr-xr-x  2 root root 0 Apr  5 12:50 init.scope
-rw-r--r--  1 root root 0 Apr  5 12:51 io.cost.model
-rw-r--r--  1 root root 0 Apr  5 12:51 io.cost.qos
-rw-r--r--  1 root root 0 Apr  5 12:51 io.pressure
-rw-r--r--  1 root root 0 Apr  5 12:51 io.prio.class
-r--r--r--  1 root root 0 Apr  5 12:51 io.stat
-r--r--r--  1 root root 0 Apr  5 12:51 memory.numa_stat
-rw-r--r--  1 root root 0 Apr  5 12:51 memory.pressure
-r--r--r--  1 root root 0 Apr  5 12:51 memory.stat
-r--r--r--  1 root root 0 Apr  5 12:51 misc.capacity
drwxr-xr-x  2 root root 0 Apr  5 12:51 proc-sys-fs-binfmt_misc.mount
drwxr-xr-x  2 root root 0 Apr  5 12:51 sys-fs-fuse-connections.mount
drwxr-xr-x  2 root root 0 Apr  5 12:51 sys-kernel-config.mount
drwxr-xr-x  2 root root 0 Apr  5 12:51 sys-kernel-debug.mount
drwxr-xr-x  2 root root 0 Apr  5 12:51 sys-kernel-tracing.mount
drwxr-xr-x 28 root root 0 Apr  5 12:52 system.slice
drwxr-xr-x  3 root root 0 Apr  5 12:52 user.slice
```

Because cgroups are structured hierarchically this isn't the best overview option.

Prettier one can be seen using:

```bash
root@ubuntu-s-1vcpu-2gb-fra1-01:~# systemd-cgls
Control group /:
-.slice
├─user.slice
│ └─user-0.slice
│   ├─session-1.scope
│   │ ├─7681 sshd: root@pts/0
│   │ ├─7785 -bash
│   │ ├─8086 systemd-cgls
│   │ └─8087 pager
│   └─user@0.service …
│     └─init.scope
│       ├─7690 /lib/systemd/systemd --user
│       └─7691 (sd-pam)
├─init.scope
│ └─1 /sbin/init
└─system.slice
  ├─packagekit.service
  │ └─1652 /usr/libexec/packagekitd
  ├─systemd-networkd.service
  │ └─544 /lib/systemd/systemd-networkd
  ├─systemd-udevd.service
  │ └─598 /lib/systemd/systemd-udevd
....
```

2. To see cgroups in action we can span a new Docker container:

```bash
root@ubuntu-s-1vcpu-2gb-fra1-01:~# docker run --rm -d alpine sleep 100
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
4abcf2066143: Pull complete
Digest: sha256:c5b1261d6d3e43071626931fc004f70149baeba2c8ec672bd4f27761f8e1ad6b
Status: Downloaded newer image for alpine:latest
8db7022657c7635747bf80db68ca9f27f49267d38c662ec976e1ee308758ad77
```

```bash
root@ubuntu-s-1vcpu-2gb-fra1-01:~# systemd-cgls
Control group /:
-.slice
├─user.slice
│ └─user-0.slice
│   ├─session-1.scope
│   │ ├─7681 sshd: root@pts/0
│   │ ├─7785 -bash
│   │ ├─9579 systemd-cgls
│   │ └─9580 pager
│   └─user@0.service …
│     └─init.scope
│       ├─7690 /lib/systemd/systemd --user
│       └─7691 (sd-pam)
├─init.scope
│ └─1 /sbin/init
└─system.slice
  ├─containerd.service …
  │ ├─8772 /usr/bin/containerd
  │ └─9528 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 8db7022657c7635747bf80db68ca9f27f49267d38c662ec976e1ee308758ad77 -address /run/containerd/containerd.sock
  ├─system-serial\x2dgetty.slice
  │ └─serial-getty@ttyS0.service
  │   └─754 /sbin/agetty -o -p -- \u --keep-baud 115200,57600,38400,9600 ttyS0 vt220
  ├─docker.service …
  │ └─9236 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
  ├─polkit.service
  │ └─1656 /usr/libexec/polkitd --no-debug
  ├─networkd-dispatcher.service
  │ └─706 /usr/bin/python3 /usr/bin/networkd-dispatcher --run-startup-triggers
  ├─multipathd.service
  │ └─378 /sbin/multipathd -d -s
  ├─systemd-journald.service
  │ └─336 /lib/systemd/systemd-journald
  ├─unattended-upgrades.service
  │ └─9422 /usr/bin/python3 /usr/share/unattended-upgrades/unattended-upgrade-shutdown --wait-for-signal
  ├─ssh.service
  │ └─1277 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
  ├─snapd.service
  │ └─709 /usr/lib/snapd/snapd
  ├─rsyslog.service
  │ └─708 /usr/sbin/rsyslogd -n -iNONE
  ├─docker-8db7022657c7635747bf80db68ca9f27f49267d38c662ec976e1ee308758ad77.scope …
  │ └─9679 sleep 100
  ├─systemd-resolved.service
```

According to the output above this container cgroup will be located inside _system.slice_.

```bash

root@ubuntu-s-1vcpu-2gb-fra1-01:~# ls -l /sys/fs/cgroup/system.slice/docker-8db7022657c7635747bf80db68ca9f27f49267d38c662ec976e1ee308758ad77.scope/
total 0
-r--r--r-- 1 root root 0 Apr  5 12:56 cgroup.controllers
-r--r--r-- 1 root root 0 Apr  5 12:56 cgroup.events
-rw-r--r-- 1 root root 0 Apr  5 12:56 cgroup.freeze
--w------- 1 root root 0 Apr  5 12:57 cgroup.kill
-rw-r--r-- 1 root root 0 Apr  5 12:57 cgroup.max.depth
-rw-r--r-- 1 root root 0 Apr  5 12:57 cgroup.max.descendants
-rw-r--r-- 1 root root 0 Apr  5 12:56 cgroup.procs
-r--r--r-- 1 root root 0 Apr  5 12:57 cgroup.stat
-rw-r--r-- 1 root root 0 Apr  5 12:56 cgroup.subtree_control
-rw-r--r-- 1 root root 0 Apr  5 12:57 cgroup.threads
-rw-r--r-- 1 root root 0 Apr  5 12:56 cgroup.type
-rw-r--r-- 1 root root 0 Apr  5 12:57 cpu.idle
-rw-r--r-- 1 root root 0 Apr  5 12:56 cpu.max
-rw-r--r-- 1 root root 0 Apr  5 12:57 cpu.max.burst
-rw-r--r-- 1 root root 0 Apr  5 12:57 cpu.pressure
-r--r--r-- 1 root root 0 Apr  5 12:56 cpu.stat
-rw-r--r-- 1 root root 0 Apr  5 12:57 cpu.uclamp.max
-rw-r--r-- 1 root root 0 Apr  5 12:57 cpu.uclamp.min
-rw-r--r-- 1 root root 0 Apr  5 12:56 cpu.weight
-rw-r--r-- 1 root root 0 Apr  5 12:57 cpu.weight.nice
-rw-r--r-- 1 root root 0 Apr  5 12:56 cpuset.cpus
-r--r--r-- 1 root root 0 Apr  5 12:57 cpuset.cpus.effective
-rw-r--r-- 1 root root 0 Apr  5 12:57 cpuset.cpus.partition
-rw-r--r-- 1 root root 0 Apr  5 12:56 cpuset.mems
-r--r--r-- 1 root root 0 Apr  5 12:57 cpuset.mems.effective
-r--r--r-- 1 root root 0 Apr  5 12:57 hugetlb.2MB.current
-r--r--r-- 1 root root 0 Apr  5 12:57 hugetlb.2MB.events
-r--r--r-- 1 root root 0 Apr  5 12:57 hugetlb.2MB.events.local
-rw-r--r-- 1 root root 0 Apr  5 12:57 hugetlb.2MB.max
-r--r--r-- 1 root root 0 Apr  5 12:57 hugetlb.2MB.rsvd.current
-rw-r--r-- 1 root root 0 Apr  5 12:57 hugetlb.2MB.rsvd.max
...
```

Inside this folder there are multiple interesting files:

- memory.max : limit memory usage for all processes inside cgroup
- cpu.max: limit CPU
- pids.max: limit number of processes it can be spawned (prevent forkbom)
- cgroup.procs: which processes are assigned to this cgroup

To check max memory:

```bash
cat memory.max
max
```

Inside cgroup.

3. Span a new container with memory limit set.

```bash
docker run --memory 50M --rm -d alpine sleep 100
```

```bash
root@ubuntu-s-1vcpu-2gb-fra1-01:/# cat /sys/fs/cgroup/system.slice/docker-c6e5965511de6ffcffe2e778491d3d63ab244a30084ca74224bf58c55894bb6e.scope/memory.max
52428800
```

4. Because this way of checking is quite cumbersome we can also use CLI:

```bash
root@ubuntu-s-1vcpu-2gb-fra1-01:/# systemctl show docker-c6e5965511de6ffcffe2e778491d3d63ab244a30084ca74224bf58c55894bb6e.scope
TimeoutStopUSec=1min 30s
Result=success
RuntimeMaxUSec=infinity
Slice=system.slice
ControlGroup=/system.slice/docker-c6e5965511de6ffcffe2e778491d3d63ab244a30084ca74224bf58c55894bb6e.scope
MemoryCurrent=270336
MemoryAvailable=52158464
CPUUsageNSec=39694000
EffectiveCPUs=0
EffectiveMemoryNodes=0
TasksCurrent=1
....
MemoryHigh=infinity
MemoryMax=52428800
MemorySwapMax=52428800
MemoryLimit=infinity
DevicePolicy=strict
DeviceAllow=char-pts rwm
```
