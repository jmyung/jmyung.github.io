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
```

```sh
buildah bud --no-cache -t docker.io/human537/buildah:v3 .
buildah push docker.io/human537/buildah:v3
```

> history : [link](https://quay.io/repository/buildah/stable/manifest/sha256:a8ea8f5de48b6285d3ac6e6a4ef5d4f0d2f0cbd7c142dcd5c90752794ba2da05)

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
