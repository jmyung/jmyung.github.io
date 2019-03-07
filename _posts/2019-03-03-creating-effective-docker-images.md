---
title: "Creating Effective Docker Images"
date: 2019-03-03 23:50:31 -0400
categories: cloud
---
# 효율적으로 도커 이미지 생성하기

## 1. 레이어는 어떻게 작동하는가?

### 1-1. 컨테이너 레이어 란

먼저 도커의 기본 개념인 **컨테이너 레이어** 에 잠시 살펴보면, 다음과 같은 구조로 이루어져 있습니다.

[![01](https://docs.docker.com/storage/storagedriver/images/container-layers.jpg)]()

- **이미지 레이어 (Image layers)** : Read-only 컨테이너 레이어 라고도 불리며, 말 그대로 읽는 것 이외에는 아무것도 할 수 없습니다.
- **컨테이너 레이어 (Container layer)** : 이미지 레이어들의 최상단에 위치하며, 매우 얇은 R/W 레이어가 위치합니다. 도커 빌드 시, 해당 레이어에서 인스톨, 실행 등의 명령을 수행합니다.

### 1-2. 왜 많은 레이어들에 신경써야 하는가?

- 레이어가 많아지면 도커 이미지도 커집니다.
- 이미지가 커지면
  - 빌드하는데 시간이 오래 걸립니다.
  - 도커 레지스트리로부터 push / pull 하는데 시간이 오래 걸립니다.
- 반대로 이미지가 작아지면
  - 빌드와 디플로이 시간이 빨라집니다.
  - 보안 취약(vulnerabilities) 관점에서 공격받을 수 있는 표면(attack surface)가 작아집니다.



## 2. 레이어를 어떻게 줄일 수 있을까?

[![01](https://media.giphy.com/media/5TcBCVA1xlJv2/giphy.gif)]()

도커 이미지의 컨테이너 레이어를 어떻게 줄일 수 있을까요?

- 가능하면 공유된 베이스 이미지를 사용 : 파이썬을 사용하는 경우, 파이썬 의존성을 가지고 있는 하나의 베이스 이미지를 사용합니다.
- 컨테이너 레이어에 데이터를 쓰지 않도록 합니다. : 예를 들면 인스톨 명령을 수행하는 경우, 불필요하게 사용하지 않는 인스톨 데이터가 남아 있을 수 있으므로, 동일 레이어에서 커밋하기 전에 제거합니다.
- RUN 을 묶습니다.
- 가능하면 캐시가 깨지지 않도록 합니다. 내가 캐시를 깨트릴때까지, 도커 캐시 사용을 미루도록 합니다. 캐시를 깬 이후에는 다시 빌드해야합니다.

## 3. 작은 이미지 만들기 (기초)

도커파일 (Dockerfile) : 이미지를 만드는데 필요한 일련의 명령어 모음이 들어있는 파일로, 컨테이너 이미지를 만들기 위해서는 기본적으로는 이하 3가지만 알아두면 됩니다.
- RUN
- ADD
- COPY

### 3-1. 베이스 이미지의 크기가 중요합니다.

- Dockerfile
```
FROM ubuntu:latest
LABEL maintainer jesang.myung@samsung.com
RUN apt-get update --y && apt-get install --y python-pip python-dev build-essential
COPY . /all
WORKDIR /app
RUN pip install -r requirement.txt
EXPOSE 5000
ENTRYPOINT ["python"]
CMD ["application.py"]
```

ubuntu는 베이스 이미지로 read-only 레이어 입니다. 다음으로 내가 추가한 명령어들이 그 위에 다른 레이어로 올라가게 되며, 이것이 최종 이미지 사이즈의 크기를 결정하게 됩니다.

### 3-2. 개선해 봅시다.

#### First Step : 적절한 베이스 이미지 선택하기

```sh
[~/dev/jmyung.github.io]$ docker pull alpine
[~/dev/jmyung.github.io]$ docker pull alpine
[~/dev/jmyung.github.io]$ docker images
REPOSITORY       TAG          IMAGE ID            CREATED             SIZE
ubuntu           latest       47b19964fb50        4 weeks ago         88.1MB
alpine           latest       caf27325b298        5 weeks ago         5.53MB
```

빌드시 사이즈가 작은 베이스 이미지를 선택할 수 있습니다. alpine이 대표적인 이미지로 ubuntu가 90MB 인 것에 비해, alpine은 겨우 5MB 남짓입니다. 최적화 같은 것을 고려하지 않고 alpine 베이스 이미지 위에 내가 추가한 명령어들로 새로운 이미지를 만든다면, 디스크 공간을 줄일 수 있습니다. MB 단위가 크기 와닿지 않는다면, 예를 들어 2GB 이미지를 사용하고 CI/CD 환경에서 200개의 배포를 수행한다고 가정하면 총 400GB의 디스크 공간이 필요합니다. 또한, 스케일링 업하는 경우 항상 얼마 만큼의 디스크 공간을 사용하는지 염두해야 합니다.

```
FROM ubuntu:latest
LABEL maintainer jesang.myung@gmail.com  # 이메일 수정
RUN apt-get update --y && apt-get install --y python-pip python-dev build-essential
COPY . /all
WORKDIR /app
RUN pip install -r requirement2.txt # 텍스트 파일 변경
EXPOSE 5000
ENTRYPOINT ["python"]
CMD ["application.py"]
```
도커파일로 돌아가서 이메일을 변경했다던지, 라인에 주석을 달거나 명령어를 수정하는 경우에도 매번 빌드시 디스크가 부족해질 수 있습니다.

#### Slightly better : 다른 배포 선택하기

```sh
[~/dev/jmyung.github.io]$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
golang              latest              cee68f119e19        2 days ago          772MB
python              latest              32260605cf7a        2 days ago          929MB
ruby                latest              616c3cf5968b        2 days ago          870MB
debian              latest              0af60a5c6dd0        2 days ago          101MB
ubuntu              latest              47b19964fb50        4 weeks ago         88.1MB
alpine              latest              caf27325b298        5 weeks ago         5.53MB
```
