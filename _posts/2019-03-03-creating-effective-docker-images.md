---
title: "Creating Effective Docker Images"
date: 2019-03-03 23:50:31 -0400
categories: cloud
---
# 효율적으로 도커 이미지 생성하기

## 1. 레이어는 어떻게 작동하는가?

### 1-1. 컨테이너 레이어 란

먼저 도커의 기본 개념인 **컨테이너 레이어** 에 대해 알아 보겠습니다.

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
