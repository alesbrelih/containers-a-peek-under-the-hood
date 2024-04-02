# Creating isolated processes using namespaces

1. Search for information about namespaces

```bash
man namespaces
```

2. Isolating process hostname from the host by spawning shell inside a different namespace.

```bash
unshare --uts sh
```

3. Checking hostname

```sh
# hostname
k8s
```

4. Changing hostname

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
