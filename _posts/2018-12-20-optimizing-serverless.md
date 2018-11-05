---
title: "Optimizing Serverless Application"
date: 2018-12-20 09:40:30 -0400
categories: cloud
---
# 서버리스 어플리케이션 최적화 해보기

서버리스의 단점 중의 하나인 **콜드 스타트 지연(Cold Start delay)** 에 대한 얘기를 해보고자 합니다.

여기서 잠깐 서버리스의 개념을 구분해보겠습니다.

- **서버리스 (Serverless)** : 서버를 고려하지 않고 애플리케이션과 서비스를 구축하고 실행하는 개념적인 의미
- **서버리스 함수 (Serverless Functions)** : 서버리스 사상으로 커스텀한 함수를 실행시킬 수 있도록 하는 것. 서버를 프로비저닝하거나 관리할 필요 없이 코드를 실행할 수 있음.
- **FaaS (Function as a Service)** : Serverless Functions를 각 클라우드 프로바이더가 각 환경에 적합한 서비스로 제공하는 것

## 1. 콜드 스타트 지연(Cold Start delay)

서버리스는 이벤트 중심이므로 코드가 항상 실행되지 않습니다. 다음과 같은 순서로 실행이 되는데요.

1. 사용자 요청 이벤트 발생
2. 관련 함수를 찾음
3. 패키지 압축해제
4. 컨테이너 로드
5. 실행가능한 상태로 만드는 서비스를 트리거

> 위 절차는 최대 수백 ms 가 소요되며, 이를 콜드 스타트 지연(Cold Start delay)이라고 합니다.

관심이 있던 차에 이에 대한 몇가지 해결 방법을 찾아보았습니다.

#### 1-1. 첫번째 방법 : 몇 분 간격으로 함수에 핑을 날리기

구글링 해보면 대부분의 해결책은 이것으로 귀결됩니다.
5분마다 한 번씩 주기적으로 함수를 호출하는 서비스를 스케줄링하여 warm 상태로 유지합니다. 이 방법은 추가 비용이 발생하는 단점이 있습니다.

[![opt01](https://cdn-images-1.medium.com/max/1600/1*qIb1MTRQymx1a-Z7uvC9lg.png)]()

참고 : https://medium.com/@sanghee/aws-%ED%98%84%EC%9E%AC-%EC%83%81%ED%83%9C-%ED%8C%8C%EC%95%85%ED%95%98%EA%B8%B0-7e82e8a1a27f

#### 1-2. 두번째 방법 : 더 많은 메모리 할당

더 많은 메모리를 예약하도록 함수를 구성하면, 빠른 시작과 향상된 성능을 제공합니다. 또한, 프로그래밍 언어도 중요한 요소로 자바는 python, nodejs보다 콜드 스타트 시간이 더 깁니다.

#### 1-3. 세번째 방법 : 캐싱

캐싱기능을 이용하면 압축된 패키지를 해제할 필요없이 바로 컨테이너에 로드할 수 있습니다.

여기까지가 찾아보면 많이 알 수 있는 내용입니다.
**AWS re:Invent 2018** 에서도 서버리스 어플리케이션 최적화에 다뤘었는데요. 람다 사용시 최적화와 관련된 내용을 간략하게 정리해 봤습니다.

### 서버리스 함수 어플리케이션 구조

[![opt01](https://github.com/jmyung/jmyung.github.io/blob/master/assets/images/lambda_opt_01.png?raw=true)]()

서버리스 함수는 위와 같은 구조를 가지고 있습니다.


### 람다 펑션 해부

[![opt03](https://github.com/jmyung/jmyung.github.io/blob/master/assets/images/lambda_opt_03.png?raw=true)]()

펑션을 해부해보면 위와 같은 네겹의 껍질로 구분되는데요.

- Compute substrate : 람다가 도는 HW. Firecracker
- Execution Environment : 실행 환경 (ex eni, VPC 등 변수들)
- Language runtime : JVM 최적화. 파이썬, Go는 바이너리
- Your function : 스택의 최상단에 위치. 내 함수를 최적화할 수 있다

이중에 Execution Environment, Your function 두 영역에 대해 얘기해보겠습니다

## 2. 함수 (function) 영역의 최적화

### 람다 컴포넌트

모든 람다는 3개의 메인 컴포넌트가 있습니다.

- Handler() function
 - 호출시 실행
 - 비즈니스 로직

- Event object
 - 펑션 호출시 이벤트 트리거

- Context object
 - 비즈니스 로직과 무관
 - 실행위치, 타임아웃, 메모리 설정, 로그 그룹 등 관련

다음의 파이선 핸들러 함수를 보시면

[![opt05](https://github.com/jmyung/jmyung.github.io/blob/master/assets/images/lambda_opt_05.png?raw=true]()

subFunctionA, subFunctionB에 비즈니스 로직을 담게 됩니다.

#### 2-1. 초기화

이때, DB나 외부 서비스를 이용한다고 하면 다음과 같이 몇 줄이 추가됩니다.

[![opt06](https://github.com/jmyung/jmyung.github.io/blob/master/assets/images/lambda_opt_06.png?raw=true)]()

> 이때 초기화를 선별하여 로드하는 것을 권장합니다.

위에 붉은 박스에서는 초기화를 담당합니다. 라이브러리를 가져오고, db 커넥션 관련 시크릿을 가져오고, db와 연결합니다. 이때, 만약 초기화 부분을 핸들러 (myhandler) 함수 안에 넣게 되면, 매번 로직을 수행할 때 마다 초기화를 수행합니다. 사실 붉은 박스는 persistent하기에 컨테이너가 초기화될때만 수행됩니다. 람다 함수가 제일 처음 실행될 때는 윗 부분부터 마지막까지 순서대로 실행되기 때문입니다. 그러나, 같은 컨테이너 내에서 두번째 람다 함수가 실행될 때는 핸들러 (myhandler) 함수만 실행됩니다. 컨테이너가 계속 떠있는 것을 warm 상태라고 하는데 람다의 경우에는 5~15분 동안 warm 상태를 유지하고, 이후에는 cold 상태로 변경이 됩니다.

여기서 착안점은 모든 초기화를 로드할 필요가 없는 경우 로드하지 말라는 것입니다. 필요 이상으로 로드하는 경우 성능에 영향을 줍니다.

#### 2-2. 비즈니스 로직 분리

[![opt07](https://github.com/jmyung/jmyung.github.io/blob/master/assets/images/lambda_opt_07.png?raw=true)]()

> 다음으로는 비즈니스 로직을 핸들러와 분리해야합니다.

실행할 때 모든 경로를 타지 않기 때문에 모든 경로의 비즈니스 로직은 로드될 필요가 없습니다. 핸들러를 entry point로 하고 비즈니스 로직을 분리하면 실행 속도를 더 높일 수 있습니다.

## 3. Execution Environment 영역의 최적화

[![opt08](https://github.com/jmyung/jmyung.github.io/blob/master/assets/images/lambda_opt_08.png?raw=true)]()

두번째로 Execution Environment 영역에서의 최적화를 생각해보겠습니다.

### 서버리스 함수의 라이프사이클

[![opt09](https://github.com/jmyung/jmyung.github.io/blob/master/assets/images/lambda_opt_09.png?raw=true)]()

두 개의 세그먼테이션으로 나뉘는데, 왼쪽은 클라우드 환경에서 최적화를 할 수 있는 영역이고, 오른쪽은 개발자가 펑션이나 코드 측면에서 최적화할 수 있는 영역입니다.

#### 3-1. 적절한 메모리 할당하기

람다 설정에서 컴퓨팅 리소스를 할당받는 방법은 `메모리` 양을 선택하는 것 외에는 없습니다. 메모리를 선택하면 이에 비례하여 CPU용량과 기타 리소스가 할당됩니다. 메모리를 늘리면 더 많은 CPU를 할당받을 수 있습니다.

람다의 과금정책은

- 함수 요청 수
- 코드 실행시간

에 따라 부과됩니다.

> 이때, 적절한 메모리 셋팅을 통해 과금대비 최적의 성능을 내는 구간을 찾아볼 수 있습니다.

[![opt10](https://github.com/jmyung/jmyung.github.io/blob/master/assets/images/lambda_opt_10.png?raw=true)]()

위 예시는 1000000 이하의 소수(prime numbers)를 1000번 반복/계산하는 함수를 수행한 결과, 발생한 시간과 비용입니다.

[![opt11](https://github.com/jmyung/jmyung.github.io/blob/master/assets/images/lambda_opt_11.png?raw=true)]()

128MB의 메모리 리소스를 사용하는 구간에서는 11초가 소요됐고, 1024MB 메모리를 사용한 경우에는 1.4초로 약 10배가량 시간이 단축됐습니다. 이에 반해 증가한 비용은 겨우 $0.00001 (0.011원)입니다. 이처럼 나의 펑션에 맞는 리소스 할당을 찾는 것도 최적화의 한 방법이 될 수 있습니다.


#### 3-2. 이외에도

- MSA화할때 모든 서비스를 API로 노출시킬 필요가 있는가?
- Concurrency 관리
- 네트워크 위치 (VPC를 사용할 것인가 말것인가, )

등에 대해서도 다뤘습니다.

이하는 위에서 설명한 내용들을 영상으로 보실 수 있습니다.

[![aws](http://img.youtube.com/vi/sSSMTSn2xmA/0.jpg)](https://youtu.be/sSSMTSn2xmA)
