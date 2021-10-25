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

priviledge mode 사용

yaml
```yaml
apiVersion: v1 
kind: Pod 
metadata: 
    name: dind 
spec: 
    containers: 
      - name: docker-cmds 
        image: docker:1.12.6 
        command: ['docker', 'run', '-p', '80:80', 'httpd:latest'] 
        resources: 
            requests: 
                cpu: 10m 
                memory: 256Mi 
        env: 
          - name: DOCKER_HOST 
            value: tcp://localhost:2375 
      - name: dind-daemon 
        image: docker:1.12.6-dind 
        resources: 
            requests: 
                cpu: 20m 
                memory: 512Mi 
        securityContext: 
            privileged: true 
        volumeMounts: 
          - name: docker-graph-storage 
            mountPath: /var/lib/docker 
    volumes: 
      - name: docker-graph-storage 
        emptyDir: {}
```

```sh
/ # ps -ef | grep docker
    1 root       0:00 docker run -p 80:80 httpd:latest
   37 root       0:00 grep docker

/ # whoami
root

/ # mkdir workspace

/ # cd workspace/

/workspace # vi Dockerfile

/workspace # docker build -t ghcr.io/jmyung/test:v11 .
...
Step 5 : CMD echo Image created
 ---> Running in 4e726984505b
 ---> 4d94b19bb349
Removing intermediate container 4e726984505b
Successfully built 4d94b19bb349
/workspace # docker images
REPOSITORY            TAG                 IMAGE ID            CREATED             SIZE
ghcr.io/jmyung/test   v11                 4d94b19bb349        8 seconds ago       163.1 MB
httpd                 latest              1132a4fc88fa        3 days ago          143.5 MB
ubuntu                latest              ba6acccedd29        9 days ago          72.77 MB
```


