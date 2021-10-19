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
