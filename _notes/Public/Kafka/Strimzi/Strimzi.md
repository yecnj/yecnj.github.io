---
title: Strimzi를 통한 커넥트 클러스터 및 커넥터 관리
feed: show
date: 23-04-2025
permalink: /strimzi-connect-and-connector
render_with_liquid: false
---

> 배경 설명은 앞선 포스트 [[Kafka 파이프라인 플레이메모]]를 참고해주세요.

## 사용한 기술

- Strimzi 0.45.0
- ArgoCD v2.9.5
- Helm 3

## 목적
- 커넥트 클러스터와 커넥터를 쉽고 빠르게 배포할 수 있는 구성 마련
- 커넥터 문제 발생 시 빠르게 대응할 수 있는 구성 마련

## Strimzi 개요

![strimzi](/assets/img/kafka/strimzi_logo.png "strimzi")

<br/>

Strimzi는 Kafka 관련 리소스를 쿠버네티스 클러스터 위에 쉽게 설치하고 관리할 수 있도록 돕는 오픈소스 프로젝트입니다.

ArgoCD와 유사하게 [[Cloud Native Computing Foundation(CNCF)]]의 Incubating 프로젝트로 채택되어 활발히 발전 중입니다.

사실은 좀 폐쇄적인 성향 때문에 커뮤니티 활동이 많이 적고, 자유도가 떨어지는 것 같긴합니다.

<br/>

다만, 조금만 공부하면 유용하게 관리할 수 있습니다.

끝까지 읽어보시죠.


## strimzi-kafka-operator 설치
우선, strimzi를 사용하기 위해서는 사용중인 쿠버네티스 위에 [strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator)를 설치해야합니다.

[strimzi quickstart 가이드](https://strimzi.io/docs/operators/0.30.0/quickstart)에서는 각 Kubernetes Resource YAML 파일을 받을 수 있는 방법을 안내하고 있습니다.

저는 이미 Helm 3 + ArgoCD로 만들어진 쿠버네티스 배포 파이프라인이 있었기 때문에, 레포에 있는 [Helm Chart](https://github.com/strimzi/strimzi-kafka-operator/tree/main/helm-charts/helm3/strimzi-kafka-operator)를 사용 했습니다.

<br/>

안내에 따라 배포해보면 이런 리소스들이 생깁니다.

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

이중에는, 제 목표인 커넥트 클러스터와 커넥터의 정의도 포함되어 있습니다.

## 커넥트 클러스터 관리

여기서부터는 각각 `KafkaConnect`와 `KafkaConnector` 라는 이름의 커스텀 리소스를 사용하게 됩니다.

우선 커넥트 클러스터는 이와 같이 관리됩니다.

<div class="mermaid">
flowchart TB
    subgraph "Helm Chart"
        direction LR
        SO[strimzi-kafka-operator]
        DK[connect-cluster]
    end

    subgraph "ArgoCD"
        direction TB
        APP1[strimzi-kafka-operator App]
        APP2[connect-cluster App]
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
설명드렸듯이 앞서 배포한 strimzi-operator가 Strimzi의 CRD들과 operator 파드를 배포하는데요.

Strimzi operator 파드에서 커스텀 리소스들의 동작을 관리합니다.

다른 차트(혹은 YAML)에서 KafkaConnect 리소스를 배포하면 operator 파드가 이를 감지하고 커넥트 클러스터를 생성합니다.

<br/>

이렇게 커넥트 클러스터를 생성할 수 있게 됩니다. 아래는 실제로 생성된 커넥트 클러스터 리소스입니다.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: connect-cluster
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  replicas: 3
```

그리고 Operator에 의해서 Connect 파드 포함 여러 하위 리소스들이 생성됩니다.

![connect-cluster-resource](/assets/img/kafka/connect-cluster-resource.jpg "connect-cluster-resource")

실제로 클러스터를 어떤 설정으로 띄우냐는 커스텀 리소스의 문법이긴해서요.

암만 기본 쿠버네티스 리소스와 유사하게 생겼더라도 방심하지 않고 [Strimzi 공식 문서](https://strimzi.io/docs/operators/latest/configuring.html) 를 많이 참고해야합니다.

그런데.. 공식 문서가 무지하게 길고, 읽기 복잡합니다.

<br/>

그래서, 유용했던 몇가지 옵션만 별도의 문서로 정리해두었습니다.

[[Strimzi KafkaConnect 구성 및 설정 예시]] 입니다.

모니터링, 구성 공급자, 볼륨 등 디테일한 배포 설정이 궁금하시다면 참고해주세요.

## 커넥터 관리

### 기존 문제
기존에는 이런 식으로 커넥터를 생성하는 것이 일반적이었는데요.

```json
{
  "name": "my-connector",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:mysql://mysql:3306/mydatabase",
    "connection.user": "myuser",
    "connection.password": "mypassword",
    "poll.interval.ms": "1000",
    "poll.max.batch.size": "1000",
    "poll.max.records": "1000",
    "poll.max.bytes": "1048576",
    "table.whitelist": "mytable",
    "topic.prefix": "my-prefix-",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.storage.StringConverter",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractField$Key",
    "transforms.unwrap.field": "id"
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.unwrap.add.tombstones": "true",
  }
}
```

느낌이 오셨겠지만, 이보다 더 긴 설정이 들어가거나 똑같은 설정이 반복되었습니다.

이에 따라, **커넥터 사용으로 재사용성 증대** 라는 목표가 힘들어지고, 이어서 카프카 사용까지 꺼리게 되기도 했습니다.

### KafkaConnector with Named Templates

그래서, 저는 Strimzi의 `KafkaConnector` 리소스와 **Helm Named Templates**를 통해서 커넥터를 관리하기로 생각했습니다.

우선, 아래와 같이 자주 사용하는 커넥터에 대한 tpl 파일에 `connector.debeziumMysqlSourceConnector`를 정의합니다.

```yaml
{{- define "connector.debeziumMysqlSourceConnector" -}}

{{- $version := .connector.version }}
{{- $connectCluster := .connector.connectCluster | default "data-kafka-connect-cluster" }}
{{- $tasksMax := .connector.tasksMax | default 1 }}

{{- $instance := .connector.source.instance | lower }}
{{- $database := .connector.source.database | lower }}
{{- $table := .connector.source.table | lower }}
{{- $state := .connector.state | default "running" }}

{{- $name := printf "%smysql-%s-%s" ($instance | replace "_" "") ($database | replace "_" "") ($table | replace "_" "") }}

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  name: {{ $name }}{{ if $version }}-{{ $version }}{{ end }}
  labels:
    strimzi.io/cluster: {{ $connectCluster }}
spec:
  class: "io.debezium.connector.mysql.MySqlConnector"
  tasksMax: {{ $tasksMax }}
  state: {{ $state }}
  config:
    database.hostname: "${secretManager:{{ $instance | upper }}:HOST}"
    database.port: 3306
    database.include.list: "{{ $database }}"
    table.include.list: "{{ $database }}.{{ $table }}"
    ...
```

tpl 파일에서는 커넥터 종류 별로 반복되는 설정들을 관리합니다.

- port
- topic.creation.enable
- snapshot.locking.mode
- 컨버터
- 프로토콜
- SMT
- 실행될 커넥트 클러스터를 지정하는 strimzi.io/cluster 라벨
- ...

그리고, template 파일에서 이 Named template을 사용합니다.

```
{{- range .Values.connectors }}
---
{{- include "connector.debeziumMysqlSourceConnector" (dict "environment" $.Values.environment "connector" . ) }}
{{- end }}
```

이제 Helm 차트 내 values 파일에 이렇게 정의해주면 간단하게 커넥터가 늘어납니다.

```yaml
environment: "dev"

connectors:
  - connectorTemplate: "connector.debeziumMysqlSourceConnector"
    version: "2025032102"
    source:
    instance: "instance"
    database: "schema"
    table: "table"
  - connectorTemplate: "connector.debeziumMysqlSourceConnector"
    version: "2025042401"
    source:
    instance: "instance2"
    database: "schema2"
    table: "table2"
```

이렇게 만들어진 최종 Strimzi KafkaConnector 리소스는 이렇게 생겼습니다.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  labels:
    strimzi.io/cluster: connect-cluster
  name: instance2mysql-schema2-table2-2025042401
spec:
  class: io.debezium.connector.mysql.MySqlConnector
  config:
    ...
```

저는 템플릿화를 위해서 Helm 을 통해서 정의했는데, YAML로 쓰고 싶으신분들은 바로 정의해서 사용하셔도 됩니다.

[KafkaConnector schema reference](https://strimzi.io/docs/operators/latest/configuring.html#type-KafkaConnector-reference)를 참고해서 커넥터 설정을 추가해주시면 됩니다.

<br/>

이렇게 만들어진 커넥터 리소스는 Strimzi 플랫폼을 통해서 아래와 같이 작동합니다.

<div class="mermaid">
flowchart TB
    subgraph "Git Repository"
        CR[KafkaConnector CR<br>MySQL Source<br>BigQuery Sink]
    end

    subgraph "Kubernetes"
        direction TB
        A0[KafkaConnector]
        SO[Strimzi Operator]
        
        subgraph "Connect Cluster"
            direction TB
            
            subgraph "Connect Pod 1"
                direction TB
                W1[Worker Process]
                subgraph "Active Connectors 1"
                    C1[MySQL Source<br>Task 1]
                    C2[BigQuery Sink<br>Task 1]
                end
            end
            
            subgraph "Connect Pod 2"
                direction TB
                W2[Worker Process]
                subgraph "Active Connectors 2"
                    C3[MySQL Source<br>Task 2]
                    C4[BigQuery Sink<br>Task 2]
                end
            end
        end
    end

    subgraph "External Systems"
        direction LR
        MySQL[(MySQL DB)]
        BQ[(BigQuery)]
    end

    CR -->|Deploy by ArgoCD| A0
    SO --> |Watch| A0
    SO -->|Deploy| W1 & W2
    C1 & C3 -->|Read| MySQL
    C2 & C4 -->|Write| BQ
    
    style CR fill:#BBDEFB,color:#333
    style SO fill:#FFE0B2,color:#333
    style W1 fill:#B2DFDB,color:#333
    style W2 fill:#B2DFDB,color:#333
    style C1 fill:#C8E6C9,color:#333
    style C3 fill:#C8E6C9,color:#333
    style C2 fill:#BBDEFB,color:#333
    style C4 fill:#BBDEFB,color:#333
    style A0 fill:#FFE0B2,color:#333
    style MySQL fill:#B2DFDB,color:#333
    style BQ fill:#BBDEFB,color:#333
</div>

가장 처음 배포했던 Strimzi Operator가
- KafkaConnector CR을 감시하고 변경사항을 감지합니다.
- Connect 클러스터의 각 Worker Pod에 커넥터 태스크를 분배합니다.
- 커넥터의 상태를 모니터링하고 CR status를 업데이트합니다.

<br/>

Connect Worker Pods는
- 각 Pod는 독립적인 Worker Process를 실행합니다.
- Worker는 할당받은 커넥터 태스크를 관리합니다.
- 태스크는 Pod 간에 자동으로 분산됩니다

이 이후로 커넥터의 추가 / 삭제가 용이해 지면서, 다른 카프카 클러스터의 데이터가 필요한 경우에도 MirrorMaker 커넥터를 통해 쉽게 데이터 조직의 클러스터로 가져와서 처리할 수 있게 되었습니다.

<br/>

커넥터에서는 커넥트 클러스터와는 다르게 기억에 남는 이렇다 할 특별한 옵션은 없습니다.

각 커넥터에서 지원하는 config의 파악이 대부분이기 때문에, 이에 맞게 잘 구성하면 될 것 같습니다.

<br/>

## 끝내며

우선, 주요 목적 중 하나였던 커넥트 클러스터의 관리가 용이하게 되었습니다.

- 플러그인 형상을 이미지로 관리하게 되면서 버전관리가 용이해졌고
- 여러 카프카 클러스터가 아닌 하나의 클러스터에서 커넥터를 사용할 수 있게

개선했습니다.

![connect-cluster](/assets/img/kafka/connect-cluster.jpg "connect-cluster")

strimzi operator를 통해서 상태가 관리되기 때문에
- 쿠버네티스의 기본 기능들 (스케일링, 롤링 업데이트, 리소스 격리 등)을 그대로 활용할 수 있고
- CRD에 맞는 규격화된 양식을 따라야 하기에

체감하기 어려운 유지보수성 측면에서의 이득도 있었을 것 같습니다.

<br/>

커넥터에 대해서도, REST API 대신 GitOps 방식으로 커넥터를 삭제하거나 추가할 수 있게 되었습니다.

- 휴먼 리소스를 크게 줄이고
- Git 기반버전 관리 및 롤백
- 쿠버네티스 리소스 기반으로 커넥터를 조작

하는 등 더할나위 없이 좋은 개발 경험을 가지게 되었습니다.

저는 이미 데이터 조직에서만 ArgoCD를 통해 10개 이상의 앱을 운영하고 있어 익숙했기에 더 좋은 시너지를 낼 수 있었습니다.

<br/>

또한,
- 커넥터 일시 중지
- 태스크 수 조정
- 커넥터 버전 업데이트

등 여러 운영 시나리오에서도 Connector 설정을 조금만 수정하는 것으로 간단히 관리할 수 있게 되었습니다.

```yaml
# Connector 설정 변경 예시
- connectorTemplate: "connector.mirrorSourceConnector"
  version: "2025032401"
  # state 속성 추가 (default : running) 하여 일시중지
  state: paused
  # 기본값 2 -> 5 변경
  tasksMax: 5
  source:
    instance: "backend"
    topic: "EVENT.SUBSCRIPTION.TOPIC"
```

지금은
- Debezium MySQL Source Connector
- Debezium MongoDB Source Connector
- Kafka MirrorMaker
- BigQuery Sink Connector
- HTTP Sink Connector → [[Braze]] 동기화 용으로 템플릿을 만들어 사용

이 정도 종류의 커넥터를 운영하고, 각 커넥터에 필요한 SMT 개발도 진행하고 있습니다.

모두 오픈소스 커넥터들을 사용하고있는데, 언젠가는 직접 커넥터를 구성해보고 싶기도합니다.

<br/>

이상으로 글을 마치겠습니다.