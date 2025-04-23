---
title: Strimzi를 통한 커넥트 클러스터 및 커넥터 관리
feed: hide
date: 22-04-2025
permalink: /strimzi-connect
---

> 아직 작성 중입니다 :)

> 배경 설명은 앞선 포스트 [[Kafka 파이프라인 플레이메모]]를 참고해주세요.

## 사용한 기술

- Strimzi 0.45.0
- ArgoCD v2.9.5
- Helm 3
- Prometheus 및 Grafana

## 목적
- 커넥트 클러스터와 커넥터를 쉽고 빠르게 배포할 수 있는 구성 마련
- 커넥터 문제 발생 시 빠르게 대응할 수 있는 구성 마련
- 모니터링 환경 구성

## Strimzi 개요

![strimzi](/assets/img/kafka/strimzi_logo.png "strimzi")

Strimzi는 Kafka 관련 리소스를 쿠버네티스 클러스터 위에 쉽게 설치하고 관리할 수 있도록 돕는 오픈소스 프로젝트입니다.

ArgoCD와 유사하게 [[Cloud Native Computing Foundation(CNCF)]]의 Incubating 프로젝트로 채택되어 활발히 발전 중입니다.

사실은 좀 폐쇄적인 성향 때문에 커뮤니티 활동이 많이 적고, 자유도가 떨어지는 것 같긴합니다.

<br/>

다만, 조금만 공부하면 유용하게 관리할 수 있습니다.

끝까지 읽어보시죠.


## strimzi-kafka-operator 설치
우선, strimzi를 사용하기 위해서는 사용중인 쿠버네티스 위에 [strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator)를 설치해야합니다.

[strimzi quickstart 가이드](https://strimzi.io/docs/operators/0.30.0/quickstart)에서는 Github Release 를 통해서 각 Kubernetes Resource YAML 파일을 받을 수 있는 방법을 안내하고 있습니다.

저는 이미 Helm 3 + ArgoCD로 만들어진 쿠버네티스 배포 파이프라인이 있었기 때문에 레포에 있는 [Helm Chart](https://github.com/strimzi/strimzi-kafka-operator/tree/main/helm-charts/helm3/strimzi-kafka-operator)를 사용 했습니다.

<br/>

쿠버네티스 상에 생성된 리소스들은 아래와 같습니다.

<div class="mermaid">
flowchart TB
    subgraph "Strimzi Resources"
        CM[ConfigMaps] -->|Configure| OP[Operator Deployment]
        OP -->|Create| OC[Operator Pod]
        
        CRD[CustomResourceDefinitions]
        SA[ServiceAccount]
        CRB[ClusterRoleBinding]

        CRD -.- SA -.- CRB 
    end
    
    linkStyle 2,3 stroke-width:-1px, stroke:#ffffff

    style CRD fill:#B2D8E6,color:#000
    style OP fill:#B2D8E6,color:#000
    style SA fill:#B2D8E6,color:#000
    style CRB fill:#B2D8E6,color:#000
    style CM fill:#B2D8E6,color:#000
    style OC fill:#FFE0B2,color:#000
</div>

여러가지 리소스들이 한번에 생겨나서 당황스러우셨을텐데요.

중요한 건 쿠버네티스 리소스를 커스텀으로 정의하는 CustomResourceDefinition 들 입니다.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-kafka-cluster
spec:
  kafka:
    replicas: 1
```

이제 이런 식으로 카프카 클러스터와 다른 카프카 관련 리소스들을 Deployment / ConfigMap 처럼 간단하게 생성할 수 있습니다.

## 커넥트 클러스터 관리

배경 설명에서 이미 MSK를 사용해서 카프카 클러스터를 사용한다고 말씀드렸습니다.

여기서부터는 각각 `KafkaConnect`와 `KafkaConnector` 라는 이름의 커스텀 리소스를 사용하게 됩니다.

우선 커넥트 클러스터는 이와 같이 관리됩니다.

<div class="mermaid">
flowchart TB
    subgraph "Helm Chart"
        direction LR
        SO[strimzi-kafka-operator]
        DK[data-kafka]
    end

    subgraph "ArgoCD"
        direction TB
        APP1[strimzi-kafka-operator App]
        APP2[data-kafka App]
    end

    subgraph "Kubernetes Resources"
        direction TB
        CRD[Strimzi CRDs]
        OP[Strimzi Operator]
        KC[KafkaConnect CR]
        
        subgraph "Connect Cluster"
            P1[Connect Pod 1]
            P2[Connect Pod 2]
            P3[Connect Pod 3]
        end
    end

    SO -->|Sync| APP1
    DK -->|Sync| APP2
    CRD -->|Use| KC
    APP1 -->|Install| CRD
    APP1 -->|Deploy| OP
    APP2 -->|Create| KC
    OP -->|Manage| KC
    KC -->|Deploy| P1 & P2 & P3

    style SO fill:#B7D0F4,color:#000
    style DK fill:#B7D0F4,color:#000
    style APP1 fill:#FAD1B7,color:#000
    style APP2 fill:#FAD1B7,color:#000
    style CRD fill:#FFE0B2,color:#000
    style OP fill:#FFE0B2,color:#000
    style KC fill:#FFE0B2,color:#000
    style P1 fill:#B2D8E6,color:#000
    style P2 fill:#B2D8E6,color:#000
    style P3 fill:#B2D8E6,color:#000
</div>

앞서 배포한 strimzi-kafka-operator 와 KafkaConnect CRD를 통해서 커넥트 클러스터를 생성할 수 있게 됩니다.

실제로 클러스터를 어떤 설정으로 띄우냐는 [Strimzi 공식 문서](https://strimzi.io/docs/operators/latest/configuring.html) 를 많이 참고해야합니다.

그런데.. 공식 문서가 무지하게 길고, 읽기 복잡합니다.

<br/>

그래서, 유용했던 몇가지 옵션만 별도의 문서 [[Strimzi KafkaConnect 구성 및 설정 예시]]로 정리해두었습니다.



## 커넥터 관리


