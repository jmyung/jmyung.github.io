---
title: "Buildah in Container"
date: 2021-10-20 12:50:21 -0400
categories: cloud
---
# 컨테이너 내에서 Buildah 실행 image

## Dockerfile

```
FROM quay.io/buildah/stable:latest
RUN echo build:260000:65537 > /etc/subuid; echo build:260000:65537 > /etc/subgid;
USER build
WORKDIR /home/build
```

```sh
buildah bud --no-cache -t docker.io/human537/buildah:v3 .
buildah push docker.io/human537/buildah:v3
```

> buildah image history : [link](https://quay.io/repository/buildah/stable/manifest/sha256:a8ea8f5de48b6285d3ac6e6a4ef5d4f0d2f0cbd7c142dcd5c90752794ba2da05)

## Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: buildah
spec:
  containers:
  - name: buildah
    image: docker.io/human537/buildah:v3
    imagePullPolicy: Always
    securityContext:
      privileged: true
    args:
    - sleep
    - "1000000"
```

```
kubectl exec -it buildah /bin/bash
```

테스트 Dockerfile

```
FROM ubuntu
MAINTAINER demousr@gmail.com
RUN apt-get update
RUN apt-get install -y nginx
CMD ["echo","Image created"]
```

```sh
buildah bud --no-cache -t ghcr.io/jmyung/test:v11 .
```

```sh
[build@buildah ~]$ buildah images
REPOSITORY                 TAG      IMAGE ID       CREATED          SIZE
ghcr.io/jmyung/test        v11      49c69b667025   28 seconds ago   167 MB
docker.io/library/ubuntu   latest   ba6acccedd29   3 days ago       75.2 MB
```

```sh
buildah bud --no-cache -t ghcr.io/jmyung/test:v11 .
buildah login -u $USER -p $PASSWD ghcr.io
buildah push ghcr.io/jmyung/test:v11
```

> 다중 로그인 가능


# Docker in containerd

rootless 유저 사용

yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: rootless
spec:
  containers:
    - image: docker:20.10.9-dind-rootless
      name: rootless
      env:
        - name: DOCKER_HOST
          value: unix:///run/user/1000/docker.sock
      securityContext:
        privileged: true
```

```sh
/ $ cat /etc/passwd
...
dockremap:x:100:101:Linux User,,,:/home/dockremap:/sbin/nologin
rootless:x:1000:1000:Rootless:/home/rootless:/bin/ash

~ $ ps -eaf
PID   USER     TIME  COMMAND
    1 rootless  0:00 rootlesskit --net=vpnkit --mtu=1500 --disable-host-loopback --port-driver=builtin --copy-up=/etc --copy-up=6:2376/tcp docker-init -- dockerd --host=unix:///run/user/1000/docker.sock --host=tcp://0.0.0.0:2376 --tlsverify --tlscacert /ce-tlscert /certs/server/cert.pem --tlskey /certs/server/key.pem
   61 rootless  0:05 /proc/self/exe --net=vpnkit --mtu=1500 --disable-host-loopback --port-driver=builtin --copy-up=/etc --copy-2376:2376/tcp docker-init -- dockerd --host=unix:///run/user/1000/docker.sock --host=tcp://0.0.0.0:2376 --tlsverify --tlscacert m --tlscert /certs/server/cert.pem --tlskey /certs/server/key.pem
   72 rootless  0:03 vpnkit --ethernet /tmp/rootlesskit911312364/vpnkit-ethernet.sock --mtu 1500 --host-ip 0.0.0.0
   89 rootless  0:00 docker-init -- dockerd --host=unix:///run/user/1000/docker.sock --host=tcp://0.0.0.0:2376 --tlsverify --tlsr/ca.pem --tlscert /certs/server/cert.pem --tlskey /certs/server/key.pem
   90 rootless  0:05 dockerd --host=unix:///run/user/1000/docker.sock --host=tcp://0.0.0.0:2376 --tlsverify --tlscacert /certs/sert /certs/server/cert.pem --tlskey /certs/server/key.pem
  101 rootless  0:01 containerd --config /run/user/1000/docker/containerd/containerd.toml --log-level info
  203 rootless  0:00 sh
  269 rootless  0:00 [containerd-shim]
  556 rootless  0:00 [containerd-shim]
 1211 rootless  0:00 ps -eaf


~ $ mkdir workspace
~ $ cd workspace/
~/workspace $ vi Dockerfile
~/workspace $ docker build -t abc:v1 .
Sending build context to Docker daemon  2.048kB
Step 1/5 : FROM ubuntu
latest: Pulling from library/ubuntu
7b1a6ab2e44d: Pull complete
Digest: sha256:626ffe58f6e7566e00254b638eb7e0f3b11d4da9675088f4781a50ae288f3322
Status: Downloaded newer image for ubuntu:latest
 ---> ba6acccedd29
Step 2/5 : MAINTAINER demousr@gmail.com
 ---> Running in d405d0e136a1
...
Successfully built 6dd6b709cca4
Successfully tagged abc:v1

~/workspace $ docker images
REPOSITORY   TAG       IMAGE ID       CREATED         SIZE
abc          v1        6dd6b709cca4   4 seconds ago   163MB
ubuntu       latest    ba6acccedd29   9 days ago      72.8MB

~/.local/share/docker $ ls -al
total 56
drwx--x---   14 rootless rootless      4096 Oct 25 08:44 .
drwxr-sr-x    3 root     rootless      4096 Oct  5 17:32 ..
drwx--x--x    4 rootless rootless      4096 Oct 25 08:44 buildkit
drwx--x--x    3 rootless rootless      4096 Oct 25 08:44 containerd
drwx--x---    2 rootless rootless      4096 Oct 25 08:48 containers
drwx------    3 rootless rootless      4096 Oct 25 08:44 image
drwxr-x---    3 rootless rootless      4096 Oct 25 08:44 network
drwx--x---    6 rootless rootless      4096 Oct 25 08:48 overlay2
drwx------    4 rootless rootless      4096 Oct 25 08:44 plugins
drwx------    2 rootless rootless      4096 Oct 25 08:44 runtimes
drwx------    2 rootless rootless      4096 Oct 25 08:44 swarm
drwx------    2 rootless rootless      4096 Oct 25 08:48 tmp
drwx------    2 rootless rootless      4096 Oct 25 08:44 trust
drwx-----x    2 rootless rootless      4096 Oct 25 08:44 volumes
~/.local/share/docker $ cd overlay2/
057751b8aee876b865bfe9048b8d932f39d8334f87db273edd4ac082b0b8e088/  af2ec237c2829060f395338f9176c0b7794f5b9aa907267bbb0fd6237caed
81d32b75be2e62e5dfd2c7f7655827e3857f882fe3e0a5452e38ca4504d8effb/  l/
```
