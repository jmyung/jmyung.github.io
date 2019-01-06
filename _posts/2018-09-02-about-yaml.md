---
title: "YAML Breakdown"
date: 2018-09-02 21:04:37 -0400
categories: language
---
# 쿠버네티스를 위한 YAML 해부

쿠비쿠바 연구회용으로 작성

## 1. YAML (야믈) 이란?

- **YAML Ain't Markup Language** :
YAML은 마크 업 언어가 아닙니다.
  - **Yet Another Markup Language** : 또 다른 마크 업 언어 (처음에는..)
- 특징
  - Data Serialization Language
  - 사람이 읽기 편함
  - **애자일 랭귀지와 많이 쓰임** : 파이썬, 펄, 루비 등
  - **데이터 타입** : 맵핑, 시퀀스, 스칼라
  - 보통 **설정 파일, 데이터 저장** 용도로 많이 사용됨
    - Kubernetes 오브젝트는 주로 .yaml 포맷을 사용해 표현됨
  - JSON과 호환

## 2. 문법

확장자는 yaml 또는 yml을 사용합니다.

### 2-1. 블록 스타일

많이 사용되는 스타일로서, 가장 친숙한 형태입니다.


```yaml
host: kor-2018  # key-value mapping. :뒤에 화이트 스페이스가 꼭 붙어야함
datacenter:
  location: Seoul # Collection : 두개의 스페이스로 들여쓰기, 탭은 인식못함
  cabinet: 13
roles:
  - webserver     # List : -를 사용
  - database
```

### 2-2. 플로우 (Flow) 스타일

- JSON의 확장
- 긴줄을 합침
- 사람이 읽기 쉽지는 않음

```yaml
host: "kor-2018"
datacenter: { location: Seoul, cabinet: 13 }
roles: [ webserver , database ]
```

## 3. 자료형

### 3-1. 맵핑

key-value로 위에서 설명

### 3-2. 시퀀스

배열 또는 리스트로 위에서 설명

### 3-3. 스칼라

#### 3-3-1. 스트링 또는 숫자 표현

```yaml
host: kor-2018
datacenter:
  location: Seoul
  cabinet: "13" # " 또는 '를 사용해, non-string 을 string 으로 변경가능
roles:
  - webserver
  - database
```

#### 3-3-2. 멀티라인

| 를 사용

```yaml
host: kor-2018
datacenter:
  location: Seoul
  cabinet: "13"
roles:
  - webserver
  - database
batchtime_sch: |             # 여러 줄을 사용할 수 있다
  2018-06-30 - calculate Q2
  2018-12-31 - calculate Q4
```

| 대신 \> 를 사용하면 뉴라인을 스페이스로 인식합니다.

## 4. 구조 (Structure)

---를 사용하여 여러개의 문서를 하나의 파일에 작성 할 수 있습니다.
`쿠버네티스 디플로이먼트` 파일을 작성할 때 이렇게 많이 사용합니다.


```yaml
# Korea Host 정보 <- # 를 이용해 주석사용
---
host: kor-2018
datacenter:
  location: Seoul
  cabinet: "13"
  num: '1'
roles:
  - webserver
  - database
---
host: kor-2019
datacenter:
  location: Seoul
  cabinet: "13"
  num: '2'
roles:
  - webserver
  - database
```

## 5. 태그

! 을 사용

- URI 셋팅

```yaml
%TAG ! tag:hostdata:kor:
```

- 로컬 태그

```yaml
%TAG ! tag:hostdata:kor:
---
host: kor-2018
datacenter:
  location: !PHL Seoul # tag:hostdata:kor:PHL
  cabinet: "13"
```

- 데이터형 셋팅
  - seq - Sequence
  - map - Map
  - str - String
  - int - Integer
  - float - Floating-point decimal
  - null - Null
  - binary - Binary code
  - omap - Ordered map
  - set - Unordered set


```YAML
cabinet: !!str 13
```

## 6. 앵커 (Anchors)

- 데이터 저장 및 재활용
  - & 를 이용하여 저장
  - \* 를 이용하여 참조

```yaml
# Korea Host 정보
---
host: kor-2018
datacenter:
  location: &KOR Seoul # 저장 1
  cabinet: !!str 13
  num: !!str 1
roles: &host           # 저장 2
  - webserver
  - database
---
host: kor-2019
datacenter:
  location: *KOR       # 참조 1
  cabinet: "13"
  num: '2'
roles: *host           # 참조 2
```

## 7. 기타

> 비교 : YAML vs JSON vs XML

### YAML
```YAML
person:
  firstname: Tom
  lastname: Smith
  year: 1982
  favorites:
    - tennis
    - golf
```

### JSON

```json
{
"person": {
  "firstname": "Tom",
  "lastname": "Smith",
  "year": 1982,
  "favorites": ["tennis", "golf"]
  }
}
```

### XML

```xml
<person>
  <firstname>Tom<firstname>
  <lastname>Smith<lastname>
  <year>1982<year>
  <favorites>
    <value>tennis</value>
    <value>golf</value>
  </favorites>
</person>
```
