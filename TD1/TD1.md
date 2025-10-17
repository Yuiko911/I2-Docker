# 1.0 Setup
## Setting up your computer
On Linux, make sure to add your user to the docker group to avoid any "connect: permission denied" errors.

## Running your first container
Pulling the image :
```
$ docker pull alpine       
Using default tag: latest
latest: Pulling from library/alpine
2d35ebdb57d9: Pull complete 
Digest: sha256:4b7ce07002c69e8f3d704a9c5d6fd3053be500b7f1c69fc0d80990c2ad8dd412
Status: Downloaded newer image for alpine:latest
docker.io/library/alpine:latest

$ docker images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
alpine        latest    706db57fb206   8 days ago     8.32MB
hello-world   latest    1b44b5a3e06a   2 months ago   10.1kB
```

