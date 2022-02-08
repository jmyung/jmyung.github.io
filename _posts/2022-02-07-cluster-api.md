# Cluster API



## 1. Cluster API 개요

### Cluster API란?

- SIG Cluster Lifecycle  프로젝트
- 선언적 (declarative) 관리 제공
- 클러스터 관련 인프라 관리 (VM, 네트워크, 로드밸런서, VPC 등)

**Cluster API 목표**

- 선언적 (declarative) API를 사용하여 K8s 클러스터 라이프사이클 관리
- 다른 인프라 환경에서 동작 (cloud,  on-prem)
- 공통 오버레이션을 정의하기 위해, 기본 구현체를 제공하고, 사용자가 다른 구현체로 대체 가능
- 기능을 새로 복제해서 쓰기보다는, 기존 컴포넌트를 재사용 (예: kubeadm)

**Cluster API 용어**

- Management Cluster : Cluster API가 설치되는 K8s Cluster
- Workload (Target) Cluster : Cluster API에 의해 생성되고 관리되는 K8s Cluster
- Provider : Cluster API는 IaaS와 상호작용하기 위해 provider 사용.
  - CAPA : Cluster API Provider for AWS
  - CAPZ : Cluster API Provider for Azure
  - CAPV : Cluster API Provider for vSphere
- (new) Bootstrap cluster : Cluster를 생성하기 위한 임시 Management Cluster

### 클러스터 라이프사이클 관리를 위한 Cluster API  

클러스터 관리자 / 운영자로서, Cluster API를 이용하여 할 수 있는 것들

- 선언적으로 적합한 K8s Cluster 생성
- Cluster API가 managed 인프라 (예: AWS EC2, VPC...) / unmanaged 인프라를 관리
- Cluster scale up / down (노드 확장)
- K8s version 업그레이드
- 클러스터 삭제 (필요시, 관련 인프라 포함)

```
# mgmt cluster
kubectl get nodes

# workload cluster
kubectl get clusters

# workload cluster의 머신 정보 (control plane, machine deployment=worker node)
kubectl get machines

# worker node 정보 제공
kubectl get machinedeployment

# kubectl scale --replicas=3 machinedeployment {machinedeployment명}

```



## 2. Cluster API로 K8s 확장하기

쿠버네티스의  reconcilation loop

= control loop

reconciling desired state and actual state

### K8s API kind

```
kubectl api-resources --api-group='networking'
```

새로운 kind 를 추가하기 위해서는 CRD 사용

CRD detail을 확인하기 위해 `kubectl explain` 사용

```
kubectl get crd | grep 'cluster.x-k8s.io'
```

```
kubectl api-resources --api-group='cluster.x-k8s.io'
```

```
kubectl explain cluster
```



## 3. Cluster API Provider 를 통한 IaaS관리

### Provider 의 역할

- Cluster API는 K8s 라이프사이클 관리 뿐만 아니라, 인프라도 관리함
- 다른 인프라 provider는 CSP간에 다른 컴포넌트를 제공하고 동작도 다르게함 (예: AWS, Azure의 네트워크는 다름)
- Cluster API는 논리적인 오브젝트로서 이용될 수 있도록, 인프라별로 특화된 기능을 세부 컴포넌트로 분할

### Cluster API Provider

- Cluster API는 Provider가 여러 종류의 IaaS 플랫폼을 통합 지원하도록 제공

- Provider 종류

  - Infrastructure Provider : AWS, Azure, vSphere 플랫폼 지원

    `kubectl get crd | grep infrastructure`

  - Bootstrap Provider : 클러스터와 노드를 bootstrapping 하기 위한 다양한 방법 제공

### Cluster API Provider for AWS (CAPA)

- Infrastructure Provider for AWS
- Provider는 CRD와 controller 제공 : AWS의 VPC, subnet, load balancer 관라

### Cluster API Bootstrap Provider for kubeadm (CABPK)

- Bootstrap Provider for kubeadm
- kubeadm 이 K8s 클러스터들을 Bootstrap 하도록 제공
- CABPK 는 kubeadm 관련 오브젝트들(KubeadmControlPlane, KubeadmConfig)을 사용

Cluster API

...

## 4. Management Cluster 생성

- clusterctl : Cluster API 관련된 오퍼레이션을 를 수행하기 위한 CLI 도구

  - clusterctl init : mgmt cluster 생성

    - clusterctl init (기 구성된 클러스터 사용)
    - clusterctl init (임시 bootstrap cluster. 예 : kind 사용)

    ```
    $ kubectl get ns
    ```

    

  - clusterctl upgrade / move / config

## 5. Workload Cluster 생성

```
clusterctl config cluster [name] 
clusterctl config cluster myung --infrastructure=aws --config ./myung-cluster.yaml > myung.yaml
kubectl apply -f myung.yaml

kubectl get clusters
kubectl ma / md / awsmachine
kubectl get secret myung-kubeocnfig -o jsopath='{.data.value}' | base64 --decode


```

