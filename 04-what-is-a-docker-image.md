# What is a docker image

If you check ./03-isolating-process-ids-and-using-chroot.md the alpine filesystem reminds you of Docker image because Docker image is basically the same thing.
But it's created a bit smarter using layers so image storage can be optimized. Else if you had 5 containers using same base image you would have to download
the same base image 5 times.

1. Create a new Docker image

```bash
mkdir /tmp/myimage
cd /tmp/myimage
touch Dockerfile
```

Contents of Dockerfile:

```Dockerfile
FROM alpine
RUN touch /tmp/mysecret
```

```bash
root@k8s:/tmp/myimage# docker build -t myimage .
Sending build context to Docker daemon  2.048kB
Step 1/2 : FROM alpine
 ---> 05455a08881e
Step 2/2 : RUN touch /tmp/myfile
 ---> Running in bea739c7048f
Removing intermediate container bea739c7048f
 ---> 66bd4785c3bc
Successfully built 66bd4785c3bc
Successfully tagged myimage:latest
```

You can see the layers when creating this image. Each `RUN` command is a new layer.

2. Inspecting Docker Image.

You CAN inspect any docker image you pull (let's pretend we didn't create this image).

```bash
mkdir /tmp/archive
cd /tmp/archive
```

Save image contents locally.

```bash
docker image save myimage -o myimage.tar
```

Inspect TAR:

```bash
root@k8s:/tmp/archive# tar -xvf myimage.tar
42ed87b0ea80efdee2f95180fb3324c42f95d7bdc7dfc078650c0395b73ad227/
42ed87b0ea80efdee2f95180fb3324c42f95d7bdc7dfc078650c0395b73ad227/VERSION
42ed87b0ea80efdee2f95180fb3324c42f95d7bdc7dfc078650c0395b73ad227/json
42ed87b0ea80efdee2f95180fb3324c42f95d7bdc7dfc078650c0395b73ad227/layer.tar
66bd4785c3bc047e5979ba8318ebc756d575534fa3f8ec749f01fa2f524d613d.json
f63623aa90fdef4b7648f19dadd87e0448e3f29d644e50bf96b2f4106d9e85a6/
f63623aa90fdef4b7648f19dadd87e0448e3f29d644e50bf96b2f4106d9e85a6/VERSION
f63623aa90fdef4b7648f19dadd87e0448e3f29d644e50bf96b2f4106d9e85a6/json
f63623aa90fdef4b7648f19dadd87e0448e3f29d644e50bf96b2f4106d9e85a6/layer.tar
manifest.json
repositories
```

Turns out that these layers are saved individually inside docker image. Basically repositories don't even know which layers belong together.

So how does repo know which layers to serve us when we pull the image? It uses the `manifest.json` file.

```bash
root@k8s:/tmp/archive# cat manifest.json | jq .
[
  {
    "Config": "66bd4785c3bc047e5979ba8318ebc756d575534fa3f8ec749f01fa2f524d613d.json",
    "RepoTags": [
      "myimage:latest"
    ],
    "Layers": [
      "42ed87b0ea80efdee2f95180fb3324c42f95d7bdc7dfc078650c0395b73ad227/layer.tar",
      "f63623aa90fdef4b7648f19dadd87e0448e3f29d644e50bf96b2f4106d9e85a6/layer.tar"
    ]
  }
]
```

RepoTags identifies the image and layers list layers that belong to the image.

Let's inspect the layers.

```bash
root@k8s:/tmp/archive# cd 42ed87b0ea80efdee2f95180fb3324c42f95d7bdc7dfc078650c0395b73ad227/
root@k8s:/tmp/archive/42ed87b0ea80efdee2f95180fb3324c42f95d7bdc7dfc078650c0395b73ad227# tar -xvf layer.tar
bin/
bin/arch
bin/ash
bin/base64
bin/bbconfig
bin/busybox
bin/cat
bin/chattr
bin/chgrp
bin/chmod
bin/chown
bin/cp
bin/date
bin/dd
...
```

First layer holds Alpine base image.

```bash
root@k8s:/tmp/archive# cd f63623aa90fdef4b7648f19dadd87e0448e3f29d644e50bf96b2f4106d9e85a6/
root@k8s:/tmp/archive/f63623aa90fdef4b7648f19dadd87e0448e3f29d644e50bf96b2f4106d9e85a6# tar -xvf layer.tar
tmp/
tmp/myfile
```

Second layer holds only the DIFF.

Why is this important? Lets say you have use a secret inside one layer and then delete it in next layer, this secret will still be inside the Docker Image (be careful).
Also if you remove all clutter inside Dockerfile as last step to make image smaller it won't actually be smaller because all files you've deleted will be in in one of the previous layers.

3. Lets put layers together.

We still need a filesystem to use and now we just have layers. Docker uses overlay mount under the hood to achieve this.

```bash
root@k8s:/tmp/archive# mkdir workdir
root@k8s:/tmp/archive# mkdir changesdir
root@k8s:/tmp/archive# mkdir mntdir
```

```bash
root@k8s:/tmp/archive# mount -t overlay overlay -o lowerdir=/tmp/archive/42ed87b0ea80efdee2f95180fb3324c42f95d7bdc7dfc078650c0395b73ad227/:/tmp/archive/f63623aa90fdef4b7648f19dadd87e0448e3f29d644e50bf96b2f4106d9e85a6/,workdir=/tmp/archive/workdir/,upperdir=/tmp/archive/changesdir/ /tmp/archive/mntdir/
```

This mounts all layers together to `/tmp/archive/mntdir`.

Options:

- lowerdir: are layers that will be immutable (Docker Image layers)
- workdir: where overlay does its magic
- upperdir: any changes inside mounted FS will be persisted here
