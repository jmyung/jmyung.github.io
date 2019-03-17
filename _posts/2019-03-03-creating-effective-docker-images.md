---
title: "Creating Effective Docker Images"
date: 2019-03-03 23:50:31 -0400
categories: cloud
---
# 효율적으로 도커 이미지 생성하기

작년 dockercon 2018에서 들었던 세션 중, `도커 이미지 생성`을 주제로한 세션이 인상에 남아서 정리해봤습니다.

## 1. 레이어는 어떻게 작동하는가?

### 1-1. 컨테이너 레이어 란

먼저 도커의 기본 개념인 **컨테이너 레이어** 를 잠시 살펴보면, 다음과 같은 구조로 이루어져 있습니다.

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
- 가능하면 캐시가 깨지지 않도록 합니다. 캐시를 깬 이후에는 다시 빌드해야하는 시간이 소요됩니다.

## 3. 보다 작은 도커 이미지 만들기

- 도커파일 (Dockerfile) : 이미지를 만드는데 필요한 일련의 명령어 모음이 들어있는 파일
- RUN, ADD, COPY 등 의 명령어 사용

### 3-1. 준비

다음 두 개의 파일을 생성합니다.

- Dockerfile
```
FROM ubuntu:latest
LABEL maintainer jesang.myung@samsung.com
RUN apt-get update -y && apt-get install -y python-pip python-dev build-essential
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
EXPOSE 5000
ENTRYPOINT ["python"]
CMD ["application.py"]
```

- requirements.txt
```
mysql-connector-python
```

ubuntu는 베이스 이미지로 read-only 레이어 입니다. 다음으로 내가 추가한 명령어들이 그 위에 다른 레이어로 올라가게 되며, 이것이 최종 이미지 사이즈의 크기를 결정하게 됩니다.

### 3-2. 개선시 고려사항들

### Check 1 : 적절한 베이스 이미지 선택하기

```sh
[~/dev/jmyung.github.io]$ docker pull ubuntu
[~/dev/jmyung.github.io]$ docker pull alpine
[~/dev/jmyung.github.io]$ docker images
REPOSITORY       TAG          IMAGE ID            CREATED             SIZE
ubuntu           latest       47b19964fb50        4 weeks ago         88.1MB
alpine           latest       caf27325b298        5 weeks ago         5.53MB
```

빌드시 사이즈가 작은 베이스 이미지를 선택할 수 있습니다. alpine이 대표적인 이미지로 ubuntu가 90MB 인 것에 비해, alpine은 겨우 5MB 남짓입니다. 최적화를 고려하지 않고 alpine 베이스 이미지 위에 내가 추가한 명령어들로 새로운 이미지를 만든다면, 디스크 공간을 줄일 수 있습니다. 예를 들어 2GB 도커 이미지를 사용하고 CI/CD 환경에서 200개의 배포를 수행한다고 가정하면 총 400GB의 디스크 공간이 필요합니다. 또한, 스케일링 업하는 경우 항상 얼마 만큼의 디스크 공간을 사용하는지 염두해야 합니다.

```
FROM ubuntu:latest
LABEL maintainer jesang.myung@gmail.com  # 이메일 수정
RUN apt-get update -y && apt-get install -y python-pip python-dev build-essential
COPY . /app
WORKDIR /app
RUN pip install -r requirements2.txt # 텍스트 파일 변경
EXPOSE 5000
ENTRYPOINT ["python"]
CMD ["application.py"]
```
이메일을 변경한다던지, 라인에 주석을 달거나 명령어를 수정하는 경우, 빌드시 디스크가 부족해질 수 있습니다.

### Check 2 : 다른 배포 이미지 선택하기

```sh
[~/dev/jmyung.github.io]$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
debian              latest              0af60a5c6dd0        2 days ago          101MB
ubuntu              latest              47b19964fb50        4 weeks ago         88.1MB
alpine              latest              caf27325b298        5 weeks ago         5.53MB
golang              latest              cee68f119e19        2 days ago          772MB
python              latest              32260605cf7a        2 days ago          929MB
ruby                latest              616c3cf5968b        2 days ago          870MB
```

다른 리눅스 배포 이미지를 선택할 수도 있고, 개발 언어를 위해 디자인된 배포 이미지를 선택할 수도 있습니다. ubuntu 이미지에서 파이선 이미지로 바꿀 수 있고, debian 이미지에서 오피셜 go, ruby 이미지로 변경할 수 있습니다.

이러한 베이스 이미지 선택의 옵션들은
- 디스크 공간이 적은 경우
- 빌드 시간 절약을 위해 베이스 이미지에 많은 레이어를 입히기 싫은 경우 등

여러 가지 상황 하에서, 이미지 선택에 대한 전략을 세울 수 있습니다.

### Check 3 : 작은 사이즈 이미지 vs Full base OS 이미지

1. 보안
  - 컴플라이언스
  - 퍼블릿 깃헙 등 검증되지 않은 소스를 다운로드
  - 금융권에서 항상 우려하는 컴플라이언스 요구사항
  - 어떤 베이스 이미지를 사용하는가 (취약성)

작은 사이즈 이미지 빌드를 위해 `alpine`를 선택할 수도 있지만, `보안성`의 이유로 `Full base OS 이미지`를 사용하는 경우도 있습니다.

2. 개발 편의성
  - 개발 편의성과 관련하여, 미니멀 베이스 이미지를 사용할 때 항상 같은 종류의 패키지 매니저를 가지고 있지 않습니다.
  - 평소에 생각하지 않는 것(예 : c 라이브러리)들이 미니멀 베이스 이미지에는 없는 경우가 많습니다. 빌드에 필수적인 요소 (build essential) 미니멀 베이스 이미지에는 이런 자잘한 신경써야 하는 부분들이 많습니다.

> 공간 효율성과 이미지를 셋업하는데 얼마나 많은 일을 할 것인가 사이에 트레이드 오프가 있습니다.

### Check 4 : 도커파일 빌드

```sh
$  docker build       -t             hi-docker    .
 |<--build command-->|<--태그 플래그-->|<--태그 명-->|<--빌드 패스-->|
```

뒤에서 도커파일에 변화를 가했을 때 이미지 사이즈가 달라지는지 확인해 보겠습니다.

### Check 5 : 플래그 활용하기

이미지를 빌드할 때 최종 이미지 사이즈에 영향을 주는 몇 개의 플래그가 있습니다.

- --cache-from : 다른 이미지로 부터 캐싱
- --compress : gzip을 이용하여 빌드 컨텍스트를 압축
- --no-cache : 빌드시에 캐시를 무시
- --squash : 새 레이어들을 싱글레이어로 압축

### Check 6 : 빌드 컨텍스트란 무엇인가?

- 빌드 컨텍스트 : 현재 디렉토리 또는 지정 위치한 파일들 집합
- 해당 파일을 빌드시에 도커 데몬에게 보냄

`docker build` 수행시 다음과 같은 메세지를 보게 됩니다.

`Sending build context to Docker daemon  10.51MB`

빌드 컨텍스트가 커질수록 도커 이미지는 커집니다. 따라서, 불필요한 파일과 디렉토리는 제거해야 합니다.

### Check 7 : 캐시

선행 도커파일 명령을 기준으로 삼아, 도커는 각각의 명령줄이 캐시 버전과 매칭되는지 확인합니다.
매칭에 대한

- ADD, COPY : 파일의 체크섬을 확인합니다.
- 그 외 명령어 : 파일 체크섬은 보지 않고, 도커파일 내 명령어 문자열을 확인합니다.

- Dockerfile 1
```
FROM ubuntu:latest
RUN apt-get update
RUN apt-get install -y mongodb-server
```

- Dockerfile 2
```
FROM ubuntu:latest
RUN apt-get update
RUN apt-get install -y mongodb-server
RUN apt-get install -y nodejs
```
위 3줄은 캐시에서 가져옵니다.

캐시가 깨지면, 레이어를 다시 빌드합니다.

### Check 8 : Multi-stage Builds

```
FROM ubuntu AS build-env
RUN apt-get update && apt-get install make
ADD . /src
RUN cd /src && make

FROM busybox
COPY --from=build-env /src/build/app /usr/local/bin/app
EXPOSE 80
ENTRYPOINT /usr/local/bin/app
```

스테이지를 분할하여 하나의 스테이지에서 다른 스테이지로 아티펙트를 복사할 수 있습니다. 결과적으로 더 작은 이미지를 만듭니다.

- 참고 예제 : https://blog.alexellis.io/mutli-stage-docker-builds/

### 3-3. 이제 개선해 봅시다.

### Step 1 : 최초 Dockerfile

- Dockerfile
```
FROM ubuntu:latest
LABEL maintainer jesang.myung@samsung.com
RUN apt-get update -y
RUN apt-get install -y python-pip python-dev build-essential
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
EXPOSE 5000
ENTRYPOINT ["python"]
CMD ["application.py"]
```

- 실행
```sh
docker build -t elephant:v1 .
```

- 결과
```
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
elephant            v1                  c5989f1284e9        10 seconds ago      496MB
```


### Step 2 : RUN 명령줄을 하나로 만들기

- before
```
FROM ubuntu:latest
RUN apt-get update
RUN apt-get install -y mongodb-server
RUN apt-get install -y nodejs
```

- after
```
FROM ubuntu:latest
RUN apt-get update \
    && apt-get install -y mongodb-server \
    && apt-get install -y nodejs
```

RUN 명령어와 관련하여 3개의 레이어에서 2개로 줄어줍니다. (docker inspect로도 확인)


실습해봅시다
- Dockerfile
```
FROM ubuntu:latest
LABEL maintainer jesang.myung@samsung.com
RUN apt-get update -y && apt-get install -y python-pip python-dev build-essential
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
EXPOSE 5000
ENTRYPOINT ["python"]
CMD ["application.py"]
```

- 실행
```sh
docker build -t elephant-slim-with-combine-run:v1 .
```

- 결과
```
REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
elephant-slim-with-combine-run        v1                  e15b4e896cd7        5 seconds ago       496MB
elephant                              v1                  c5989f1284e9        8 minutes ago       496MB
```
위 예제는 이미지 사이즈 차이는 거의 나지 않으나, `docker inspect` 로 레이어 개수를 세보면, 레이어가 한 개가 줄어있는 것을 확인할 수 있습니다.

### Step 3 : 베이스 이미지 변경

`ubuntu:latest` 에서 `python:2.7-alpine` 으로 변경

- Dockerfile
```
FROM python:2.7-alpine
LABEL maintainer jesang.myung@samsung.com
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
EXPOSE 5000
ENTRYPOINT ["python"]
CMD ["application.py"]
```

- 실행
```sh
docker build -t elephant-slim-with-change-to-alpine:v1 .
```

파이선으로 디자인된 이미지이기때문에 `RUN` 명령줄을 삭제할 수 있습니다.

- 결과
```
REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
elephant-slim-with-combine-run        v1                  e15b4e896cd7        5 seconds ago       496MB
elephant-slim-with-change-to-alpine   v1                  c9e13c9e3d0f        7 minutes ago       74MB
elephant                              v1                  c5989f1284e9        8 minutes ago       496MB
```

### Step 4 : 캐시 무효화를 더 적게 발생하도록 수정

- Dockerfile
```
FROM python:2.7-alpine
LABEL maintainer jesang.myung@samsung.com
COPY requirements.txt /app/
RUN pip install -r /app/requirements.txt
COPY . /app
WORKDIR /app
EXPOSE 5000
ENTRYPOINT ["python"]
CMD ["application.py"]
```

`COPY requirements.txt /app`의 순서를 앞에 추가하여, 캐시 무효화가 더 적게 일어나도록 합니다.
`requirements.txt` 파일이 변경된 경우에만 레이어를 다시 생성합니다.

- 실행
```sh
docker build -t elephant-slim-with-invalidation:v1 .
```

### Step 5 : USER 변경은 레이어를 추가합니다.

- Dockerfile
```
FROM ubuntu:latest
RUN groupadd -r babyshark && useradd -r -g babyshark babyshark
USER babyshark
RUN apt-get update \
    && apt-get install -y mongodb-server \
    && apt-get install -y nodejs
USER root
COPY . /app
```

### 정리해보겠습니다. 그리고 몇 가지 팁...

1. 적은 개수의 레이어는 더 작은 이미지를 의미합니다.
2. 가능하면 레이어를 share 하도록 구성합니다. (From cache)
3. 베이스 이미지 전략
  - 기존 베이스 이미지를 이용할 것인가? (런타임 환경을 위해 이미 만들어진 오피셜 이미지를 사용할 것인가)
  - 아니면 새로 만들 것인가? (minimal 베이스 이미지를 선택하여 런타임 환경, 보안 등 요소를 직접 설치)
  - 위 둘은 트레이드 오프 관계
4. 공간 효율성을 위해
  - 인스톨시 recommended packages 은 제외
  - 버전 pinning : 버전을 명시적으로 기술, 새버전을 매번 가져오지 않도록 함
