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

Container log
```sh
root@ske-cicd-844fbdb55c-qbl9s:~# k logs rootless
Generating RSA private key, 4096 bit long modulus (2 primes)
......................................................................................................................................................................................................................................................++++
...........................................................................................................++++
e is 65537 (0x010001)
Generating RSA private key, 4096 bit long modulus (2 primes)
................++++
..................................++++
e is 65537 (0x010001)
Signature ok
subject=CN = docker:dind server
Getting CA Private Key
/certs/server/cert.pem: OK
Generating RSA private key, 4096 bit long modulus (2 primes)
..................................................................................................................................................................................................................................................................................................++++
......................................++++
e is 65537 (0x010001)
Signature ok
subject=CN = docker:dind client
Getting CA Private Key
/certs/client/cert.pem: OK
time="2021-11-09T08:47:44Z" level=warning msg="failed to mount sysfs, falling back to read-only mount: operation not permitted"
[WARN  tini (90)] Tini is not running as PID 1 and isn't registered as a child subreaper.
Zombie processes will not be re-parented to Tini, so zombie reaping won't work.
To fix the problem, use the -s option or set the environment variable TINI_SUBREAPER to register Tini as a child subreaper, or run Tini as PID 1.
time="2021-11-09T08:47:44.427468214Z" level=info msg="Starting up"
time="2021-11-09T08:47:44.427496652Z" level=warning msg="Running in rootless mode. This mode has feature limitations."
time="2021-11-09T08:47:44.427501350Z" level=info msg="Running with RootlessKit integration"
time="2021-11-09T08:47:44.428546724Z" level=warning msg="could not change group /run/user/1000/docker.sock to docker: group docker not found"
time="2021-11-09T08:47:44.430236551Z" level=info msg="libcontainerd: started new containerd process" pid=109
time="2021-11-09T08:47:44.430268946Z" level=info msg="parsed scheme: \"unix\"" module=grpc
time="2021-11-09T08:47:44.430276757Z" level=info msg="scheme \"unix\" not registered, fallback to default scheme" module=grpc
time="2021-11-09T08:47:44.430297088Z" level=info msg="ccResolverWrapper: sending update to cc: {[{unix:///run/user/1000/docker/containerd/containerd.sock  <nil> 0 <nil>}] <nil> <nil>}" module=grpc
time="2021-11-09T08:47:44.430306451Z" level=info msg="ClientConn switching balancer to \"pick_first\"" module=grpc
time="2021-11-09T08:47:44.438910161Z" level=info msg="starting containerd" revision=5b46e404f6b9f661a205e28d59c982d3634148f8 version=v1.4.11
time="2021-11-09T08:47:44.457332409Z" level=info msg="loading plugin \"io.containerd.content.v1.content\"..." type=io.containerd.content.v1
time="2021-11-09T08:47:44.457883507Z" level=info msg="loading plugin \"io.containerd.snapshotter.v1.aufs\"..." type=io.containerd.snapshotter.v1
time="2021-11-09T08:47:44.460190809Z" level=info msg="loading plugin \"io.containerd.snapshotter.v1.btrfs\"..." type=io.containerd.snapshotter.v1
time="2021-11-09T08:47:44.460546275Z" level=info msg="skip loading plugin \"io.containerd.snapshotter.v1.btrfs\"..." error="path /home/rootless/.local/share/docker/containerd/daemon/io.containerd.snapshotter.v1.btrfs (ext4) must be a btrfs filesystem to be used with the btrfs snapshotter: skip plugin" type=io.containerd.snapshotter.v1
time="2021-11-09T08:47:44.460571089Z" level=info msg="loading plugin \"io.containerd.snapshotter.v1.devmapper\"..." type=io.containerd.snapshotter.v1
time="2021-11-09T08:47:44.460590062Z" level=warning msg="failed to load plugin io.containerd.snapshotter.v1.devmapper" error="devmapper not configured"
time="2021-11-09T08:47:44.460597794Z" level=info msg="loading plugin \"io.containerd.snapshotter.v1.native\"..." type=io.containerd.snapshotter.v1
time="2021-11-09T08:47:44.460644827Z" level=info msg="loading plugin \"io.containerd.snapshotter.v1.overlayfs\"..." type=io.containerd.snapshotter.v1
time="2021-11-09T08:47:44.461182056Z" level=info msg="loading plugin \"io.containerd.snapshotter.v1.zfs\"..." type=io.containerd.snapshotter.v1
time="2021-11-09T08:47:44.461523472Z" level=info msg="skip loading plugin \"io.containerd.snapshotter.v1.zfs\"..." error="path /home/rootless/.local/share/docker/containerd/daemon/io.containerd.snapshotter.v1.zfs must be a zfs filesystem to be used with the zfs snapshotter: skip plugin" type=io.containerd.snapshotter.v1
time="2021-11-09T08:47:44.461540777Z" level=info msg="loading plugin \"io.containerd.metadata.v1.bolt\"..." type=io.containerd.metadata.v1
time="2021-11-09T08:47:44.461577356Z" level=warning msg="could not use snapshotter devmapper in metadata plugin" error="devmapper not configured"
time="2021-11-09T08:47:44.461587263Z" level=info msg="metadata content store policy set" policy=shared
time="2021-11-09T08:47:44.482803005Z" level=info msg="loading plugin \"io.containerd.differ.v1.walking\"..." type=io.containerd.differ.v1
time="2021-11-09T08:47:44.482836156Z" level=info msg="loading plugin \"io.containerd.gc.v1.scheduler\"..." type=io.containerd.gc.v1
time="2021-11-09T08:47:44.482915203Z" level=info msg="loading plugin \"io.containerd.service.v1.introspection-service\"..." type=io.containerd.service.v1
time="2021-11-09T08:47:44.482967771Z" level=info msg="loading plugin \"io.containerd.service.v1.containers-service\"..." type=io.containerd.service.v1
time="2021-11-09T08:47:44.482982170Z" level=info msg="loading plugin \"io.containerd.service.v1.content-service\"..." type=io.containerd.service.v1
time="2021-11-09T08:47:44.482992890Z" level=info msg="loading plugin \"io.containerd.service.v1.diff-service\"..." type=io.containerd.service.v1
time="2021-11-09T08:47:44.483002927Z" level=info msg="loading plugin \"io.containerd.service.v1.images-service\"..." type=io.containerd.service.v1
time="2021-11-09T08:47:44.483012782Z" level=info msg="loading plugin \"io.containerd.service.v1.leases-service\"..." type=io.containerd.service.v1
time="2021-11-09T08:47:44.483022533Z" level=info msg="loading plugin \"io.containerd.service.v1.namespaces-service\"..." type=io.containerd.service.v1
time="2021-11-09T08:47:44.483034350Z" level=info msg="loading plugin \"io.containerd.service.v1.snapshots-service\"..." type=io.containerd.service.v1
time="2021-11-09T08:47:44.483044898Z" level=info msg="loading plugin \"io.containerd.runtime.v1.linux\"..." type=io.containerd.runtime.v1
time="2021-11-09T08:47:44.483159724Z" level=info msg="loading plugin \"io.containerd.runtime.v2.task\"..." type=io.containerd.runtime.v2
time="2021-11-09T08:47:44.483236897Z" level=info msg="loading plugin \"io.containerd.monitor.v1.cgroups\"..." type=io.containerd.monitor.v1
time="2021-11-09T08:47:44.483622104Z" level=info msg="loading plugin \"io.containerd.service.v1.tasks-service\"..." type=io.containerd.service.v1
time="2021-11-09T08:47:44.483650820Z" level=info msg="loading plugin \"io.containerd.internal.v1.restart\"..." type=io.containerd.internal.v1
time="2021-11-09T08:47:44.483697047Z" level=info msg="loading plugin \"io.containerd.grpc.v1.containers\"..." type=io.containerd.grpc.v1
time="2021-11-09T08:47:44.483710560Z" level=info msg="loading plugin \"io.containerd.grpc.v1.content\"..." type=io.containerd.grpc.v1
time="2021-11-09T08:47:44.483720288Z" level=info msg="loading plugin \"io.containerd.grpc.v1.diff\"..." type=io.containerd.grpc.v1
time="2021-11-09T08:47:44.483730843Z" level=info msg="loading plugin \"io.containerd.grpc.v1.events\"..." type=io.containerd.grpc.v1
time="2021-11-09T08:47:44.483740721Z" level=info msg="loading plugin \"io.containerd.grpc.v1.healthcheck\"..." type=io.containerd.grpc.v1
time="2021-11-09T08:47:44.483753140Z" level=info msg="loading plugin \"io.containerd.grpc.v1.images\"..." type=io.containerd.grpc.v1
time="2021-11-09T08:47:44.483765761Z" level=info msg="loading plugin \"io.containerd.grpc.v1.leases\"..." type=io.containerd.grpc.v1
time="2021-11-09T08:47:44.483775228Z" level=info msg="loading plugin \"io.containerd.grpc.v1.namespaces\"..." type=io.containerd.grpc.v1
time="2021-11-09T08:47:44.483786149Z" level=info msg="loading plugin \"io.containerd.internal.v1.opt\"..." type=io.containerd.internal.v1
time="2021-11-09T08:47:44.483844505Z" level=warning msg="failed to load plugin io.containerd.internal.v1.opt" error="mkdir /opt/containerd: permission denied"
time="2021-11-09T08:47:44.483856059Z" level=info msg="loading plugin \"io.containerd.grpc.v1.snapshots\"..." type=io.containerd.grpc.v1
time="2021-11-09T08:47:44.483866295Z" level=info msg="loading plugin \"io.containerd.grpc.v1.tasks\"..." type=io.containerd.grpc.v1
time="2021-11-09T08:47:44.483880097Z" level=info msg="loading plugin \"io.containerd.grpc.v1.version\"..." type=io.containerd.grpc.v1
time="2021-11-09T08:47:44.483891411Z" level=info msg="loading plugin \"io.containerd.grpc.v1.introspection\"..." type=io.containerd.grpc.v1
time="2021-11-09T08:47:44.484028488Z" level=info msg=serving... address=/run/user/1000/docker/containerd/containerd-debug.sock
time="2021-11-09T08:47:44.484069451Z" level=info msg=serving... address=/run/user/1000/docker/containerd/containerd.sock.ttrpc
time="2021-11-09T08:47:44.484108487Z" level=info msg=serving... address=/run/user/1000/docker/containerd/containerd.sock
time="2021-11-09T08:47:44.484124736Z" level=info msg="containerd successfully booted in 0.046647s"
time="2021-11-09T08:47:44.492853934Z" level=info msg="parsed scheme: \"unix\"" module=grpc
time="2021-11-09T08:47:44.492868670Z" level=info msg="scheme \"unix\" not registered, fallback to default scheme" module=grpc
time="2021-11-09T08:47:44.492889588Z" level=info msg="ccResolverWrapper: sending update to cc: {[{unix:///run/user/1000/docker/containerd/containerd.sock  <nil> 0 <nil>}] <nil> <nil>}" module=grpc
time="2021-11-09T08:47:44.492897092Z" level=info msg="ClientConn switching balancer to \"pick_first\"" module=grpc
time="2021-11-09T08:47:44.493488495Z" level=info msg="parsed scheme: \"unix\"" module=grpc
time="2021-11-09T08:47:44.493534480Z" level=info msg="scheme \"unix\" not registered, fallback to default scheme" module=grpc
time="2021-11-09T08:47:44.493564446Z" level=info msg="ccResolverWrapper: sending update to cc: {[{unix:///run/user/1000/docker/containerd/containerd.sock  <nil> 0 <nil>}] <nil> <nil>}" module=grpc
time="2021-11-09T08:47:44.493580772Z" level=info msg="ClientConn switching balancer to \"pick_first\"" module=grpc
time="2021-11-09T08:47:44.570159776Z" level=warning msg="Your kernel does not support swap memory limit"
time="2021-11-09T08:47:44.570175946Z" level=warning msg="Your kernel does not support CPU realtime scheduler"
time="2021-11-09T08:47:44.570286907Z" level=info msg="Loading containers: start."
time="2021-11-09T08:47:44.573927427Z" level=warning msg="Running modprobe bridge br_netfilter failed with message: Device \"bridge\" does not exist.\nbridge                151552  1 br_netfilter\nstp                    16384  1 bridge\nllc                    16384  2 bridge,stp\nDevice \"br_netfilter\" does not exist.\nbr_netfilter           24576  0 \nbridge                151552  1 br_netfilter\nmodprobe: can't change directory to '/lib/modules': No such file or directory\n, error: exit status 1"
time="2021-11-09T08:47:44.765274112Z" level=info msg="Default bridge (docker0) is assigned with an IP address 172.17.0.0/16. Daemon option --bip can be used to set a preferred IP address"
time="2021-11-09T08:47:44.985576321Z" level=info msg="Loading containers: done."
time="2021-11-09T08:47:44.990772949Z" level=warning msg="Not using native diff for overlay2, this may cause degraded performance for building images: running in a user namespace" storage-driver=overlay2
time="2021-11-09T08:47:44.990898811Z" level=info msg="Docker daemon" commit=79ea9d3 graphdriver(s)=overlay2 version=20.10.9
time="2021-11-09T08:47:44.990988082Z" level=info msg="Daemon has completed initialization"
time="2021-11-09T08:47:45.048827280Z" level=info msg="API listen on /run/user/1000/docker.sock"
time="2021-11-09T08:47:45.052142247Z" level=info msg="API listen on [::]:2376"
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


# buildkit

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: buildkitd
  annotations:
    container.apparmor.security.beta.kubernetes.io/buildkitd: unconfined
    container.seccomp.security.alpha.kubernetes.io/buildkitd: unconfined
# see buildkit/docs/rootless.md for caveats of rootless mode
spec:
  containers:
    - name: buildkitd
      image: moby/buildkit:master-rootless
      args:
        - --oci-worker-no-process-sandbox
      readinessProbe:
        exec:
          command:
            - buildctl
            - debug
            - workers
        initialDelaySeconds: 5
        periodSeconds: 30
      livenessProbe:
        exec:
          command:
            - buildctl
            - debug
            - workers
        initialDelaySeconds: 5
        periodSeconds: 30
      securityContext:
        # To change UID/GID, you need to rebuild the image
        runAsUser: 1000
        runAsGroup: 1000
```

> 출처 : [링크](https://github.com/moby/buildkit/blob/master/examples/kubernetes/pod.rootless.yaml)

데몬 프로세스가 하나 떠 있음

```sh
PID   USER     TIME  COMMAND
    1 user      0:00 rootlesskit buildkitd --oci-worker-no-process-sandbox
   12 user      0:00 /proc/self/exe buildkitd --oci-worker-no-process-sandbox
   27 user      0:02 buildkitd --oci-worker-no-process-sandbox
```

```sh
~/.local $ ll
total 24
drwxr-sr-x    1 user     user          4096 Oct  5 23:21 .
drwxr-sr-x    1 user     user          4096 Nov  9 04:38 ..
drwxr-sr-x    3 user     user          4096 Oct  5 23:21 share
drwxr-sr-x    1 user     user          4096 Nov  9 07:52 tmp
```

> 문제는 rootless 가 containerd workers 를 지원하지 않는다는 warning 메세지가 뜸

```sh
$ kubectl logs buildkitd
time="2021-11-09T02:17:40Z" level=info msg="auto snapshotter: using overlayfs"
time="2021-11-09T02:17:40Z" level=warning msg="NoProcessSandbox is enabled. Note that NoProcessSandbox allows build containers to kill (and potentially ptrace) an arbitrary process in the BuildKit host namespace. NoProcessSandbox should be enabled only when the BuildKit is running in a container as an unprivileged user."
time="2021-11-09T02:17:40Z" level=info msg="found worker \"vp7yzk9i406y8cgeoi02bobas\", labels=map[org.mobyproject.buildkit.worker.executor:oci org.mobyproject.buildkit.worker.hostname:buildkitd org.mobyproject.buildkit.worker.snapshotter:overlayfs], platforms=[linux/amd64 linux/386]"
time="2021-11-09T02:17:40Z" level=warning msg="rootless mode is not supported for containerd workers. disabling containerd worker."
time="2021-11-09T02:17:40Z" level=info msg="found 1 workers, default=\"vp7yzk9i406y8cgeoi02bobas\""
time="2021-11-09T02:17:40Z" level=warning msg="currently, only the default worker can be used."
time="2021-11-09T02:17:40Z" level=info msg="running server on /run/user/1000/buildkit/buildkitd.sock"
```




