---
title: 쿠버네티스 환경에서 스키마 레지스트리 및 ksqlDB 구성
feed: show
date: 21-05-2025
permalink: /ksqlDB-configuration
---

본 문서에서는
- Kubernetes 환경에서 Confluent 스키마 레지스트리 및 ksqlDB를 Helm 차트를 통해 배포하고
- 스키마 레지스트리와 연동하고
- 데베지움과 함께 사용한 방식을 설명합니다.

<br/>

### 배경
- EKS
- MSK Clsuter
- Debezium 3.0.8

<br/>

### cp-helm-charts

우선, [cp-helm-charts](https://github.com/confluentinc/cp-helm-charts) 에서 제공하는 ksqlDB Helm 차트를 사용합니다.

사실 이 레포지토리는 Feb 20, 2024에 아카이브되어 [CFK (Confluent Platform for Kubernetes)](https://docs.confluent.io/operator/current/co-deploy-cfk.html) 로 이전되었는데요,

다만, 기존에 EKS 상에서 같은 패턴으로 운영중인 프로젝트들이 있었고, Confluent Platform 관련한 레퍼런스가 많지않아 계속 사용하게 되었습니다.

<br/>

### cp-schema-registry

cp-helm-charts 아래의 서브차트 (`charts/` 디렉터리) 중 cp-schema-registry 차트를 사용합니다.


```yaml
cp-schema-registry:
  enabled: true

  kafka:
    bootstrapServers: "<broker-url-01>,<broker-url-02>"
  
  configurationOverrides:
    "kafkastore.security.protocol": "SSL"
    "schema.compatibility.level": "full"

  env:
    - name: SCHEMA_REGISTRY_OPTS
      value: "-Djava.net.preferIPv4Stack=true"
```

특별한 설정은 없고, 이렇게 설정했습니다.
- kafka.bootstrapServers는 이미 있는 MSK의 브로커 주소를 입력합니다.
- kafkastore.security.protocol으로 카프카 브로커에 연결할때 SSL 프로토콜을 사용합니다.
- schema.compatibility.level 스키마 호환성 레벨을 설정합니다. 현재는 FULL 레벨로 설정하였는데요. 운영에 따라서 바뀌게 될 것 같습니다.
- SCHEMA_REGISTRY_OPTS 에서 -Djava.net.preferIPv4Stack=true 를 설정하여 IPv4 스택을 사용합니다.
    - EKS와 같은 내부환경에서 일반적인 IPv4 스택을 사용하기때문에 java.net.UnknownHostException 등의 오류가 발생할 수 있습니다.

<br/>

### cp-ksqldb-server

마찬가지로 cp-helm-charts 아래의 서브차트 (charts/ 디렉터리) 중 cp-ksqldb-server 차트를 사용했고, 특별한 설정도 없습니다.

```yaml
cp-ksql-server:
  enabled: true

  ksql:
    headless: false

  kafka:
    bootstrapServers: "<broker-url-01>,<broker-url-02>"

  cp-schema-registry:
    url: "http://cp-schema-registry.default.svc.cluster.local:80"

  configurationOverrides:
    "consumer.security.protocol": "SSL"
    "producer.security.protocol": "SSL"
    "security.protocol": "SSL"
    "ksql.streams.auto.offset.reset": "earliest"
    "auto.offset.reset": "earliest"
```

- ksql.headless 는 false 로 설정합니다.
    - true로 설정시 REST API 로 접근 불가능하고, pod에 내장된 query 파일을 통해 배포해야합니다.
    - 하지만 ksqlDB 특성 상, 쿼리를 작성하고 배포하는 타이밍이나 배포 프로세스가 복잡해질 수 있어 사용하지 않습니다.
- cp-schema-registry.url 은 스키마 레지스트리의 URL을 입력합니다.
    - 쿠버네티스 내부 도메인으로, default namespace 에서 배포하기 때문에 default.svc.cluster.local 을 사용합니다.
- kafka 로 연결할때 SSL 프로토콜을 사용합니다.
- auto.offset.reset 오프셋 리셋 정책을 earliest 로 설정합니다.

<br/>

위에서 많은 설정이 없는 것처럼 이야기했지만, 내부 쿠버네티스 혹은 배포 정책에 따라서
- nodeSelector
- Resource
- ReplicaCount
- Image

등등 여러 추가 설정이 필요할 수 있습니다.

<br/>

### Debezium Connector 설정

```yaml
key.converter: "io.confluent.connect.avro.AvroConverter"
key.converter.schema.registry.url: "http://cp-schema-registry.default.svc.cluster.local:80"
value.converter: "io.confluent.connect.avro.AvroConverter"
value.converter.schema.registry.url: "http://cp-schema-registry.default.svc.cluster.local:80"
```

데베지움 커넥터에 또한 스키마 레지스트리를 사용하기 위해 위 설정들을 추가합니다.

저는 이미 [[Strimzi를 통한 커넥트 클러스터 및 커넥터 관리]] 에서 통합 템플릿을 만들어두어, 수십개의 커넥터에 대해서도 빠르게 수정해서 배포할 수 있었습니다.

<br/>

이제 데베지움 커넥터를 사용하는 경우, Kafka상에 Key, Value 가 AVRO 형식으로 입력되게 됩니다.

![avro](/assets/img/ksqldb/avro.jpeg "avro")

<br/>

### ksqlDB 실행

우선 EKS내에 배포한 ksqlDB 서버가 EKS 바깥에서 접근할 수 있게 구성합니다.

이후, ksqlDB 서버에 접근하여 쿼리를 작성합니다.

ksqlDB 서버는 REST API 를 통해 접근하거나, MacOS 기준 아래 도커 컨테이너 명령을 통해서 대화형 CLI로 접근할 수 있습니다.

```bash
docker run --interactive --tty --rm --platform linux/amd64 \
    confluentinc/ksqldb-cli:0.29.0 \
    ksql <ksql-server-url>:80

                  ===========================================
                  =       _              _ ____  ____       =
                  =      | | _____  __ _| |  _ \| __ )      =
                  =      | |/ / __|/ _` | | | | |  _ \      =
                  =      |   <\__ \ (_| | | |_| | |_) |     =
                  =      |_|\_\___/\__, |_|____/|____/      =
                  =                   |_|                   =
                  =        The Database purpose-built       =
                  =        for stream processing apps       =
                  ===========================================

Copyright 2017-2022 Confluent Inc.

CLI v0.29.0, Server v7.8.0 located at http://<ksql-server-url>:80

WARNING: CLI and server version don't match. This may lead to unexpected errors.
         It is recommended to use a CLI that matches the server version.

Server Status: RUNNING

Having trouble? Type 'help' (case-insensitive) for a rundown of how things work!

ksql> 
```

<br/>

### ksqlDB 예시 쿼리 작성

데베지움 커넥터로 입력된 토픽을 조회해보겠습니다.

만약 데베지움 커넥터가 구성되어있지 않다면, [Confluent ksqlDB how to guide](https://docs.confluent.io/platform/current/ksqldb/how-to-guides/control-the-case-of-identifiers.html)를 따라해보는것도 좋습니다.

간단하게 stream 을 생성하고, 조회하고 삭제해보겠습니다.

```sql
ksql> create stream test_stream with (kafka_topic='<debezium-topic>', value_format='AVRO');

 Message        
----------------
 Stream created 
----------------

ksql> select * from test_stream;
+--------------------+--------------------+--------------------+--------------------+--------------------+--------------------+
|BEFORE              |AFTER               |SOURCE              |OP                  |TS_MS               |TRANSACTION         |
+--------------------+--------------------+--------------------+--------------------+--------------------+--------------------+
|null                |{"id": ...          |{VERSION=3.0.8.Final|u                   |1747213490407       |null                |

...

ksql> show streams;

 Stream Name                               | Kafka Topic                   | Key Format | Value Format | Windowed 
------------------------------------------------------------------------------------------------------------------
 TEST_STREAM                               | <debezium-topic>              | KAFKA      | AVRO         | false    
------------------------------------------------------------------------------------------------------------------

ksql> drop stream test_stream;

 Message
------------------------------------------------------------
 Source `TEST_STREAM` (topic: <debezium-topic>) was dropped.
------------------------------------------------------------
ksql> 
```

아주 간단하죠.

`show`, `create`, `drop` 등 일반적으로 사용하던 SQL 구문들을 모두 사용할 수 있습니다.

그래서인지 저는 자연스럽게 어 이것도 되나? 하며 일단 명령어들을 치면서 금방 익숙해질 수 있었습니다.

<br/>

### 스키마 레지스트리를 쓰지 않는 경우에는?
```sql
CREATE STREAM STG_SUBMIT (
    SCHEMA STRUCT<
        TYPE STRING,
        FIELDS ARRAY<STRUCT<
            TYPE STRING,
            OPTIONAL BOOLEAN,
            FIELD STRING
        >>
    >,
    PAYLOAD STRUCT<
        BEFORE STRUCT<
            user_id INT,
            event_id INT
        >,
        AFTER STRUCT<
            user_id INT,
            event_id INT
        >,
    -- ...
```

`create stream ~ with (kafka_topic = ~)` 대신 데베지움에 들어가있는 타입들을 모두 열거해줘야합니다.

CDC 기반 데이터 처리 를 ksqlDB로 계획하고 있으시다면 가능한 스키마 레지스트리를 사용하는 것을 권장합니다.
