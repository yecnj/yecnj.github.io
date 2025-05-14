---
title: ksqlDB를 통한 스트리밍 파이프라인 구성
feed: hide
date: 23-04-2025
permalink: /ksqlDB
---

> 글을 작성하는 중입니다 :)

## ksqlDB를 통한 스트리밍 파이프라인 구성

![ksqldb](/assets/img/ksqldb/logo.png "ksqldb")

<br/>

## 글에 들어가기 앞서
이런 개념을 알고 읽으시면 좋습니다.
- SQL 사용한 데이터처리
- 큐, 카프카의 토픽, offset의 개념
- kafka 상의 파티션이 어떻게 작동하는지 간단한 개념
- 데이터 소스로 사용할 데베지움 혹은 CDC

<br/>

아래는 이 글에서 원활한 설명을 위해서 다루지 않는 내용입니다.
- 카프카 상의 파티션과 ksqlDB 상에서 사용하는 파티션에 대한 깊은 개념
- 타임 윈도우 기반의 스트림 만들기
- emit final / changes

<br/>

## ksqlDB에 대해서

ksqlDB는 SQL을 기반으로 카프카 기반 스트림 처리를 도와주는 오픈소스 프로젝트입니다.

국내에서는 [ksqlDB를 활용한 증권사의 실시간 데이터 처리하기](https://toss.tech/article/ksqldb-realtime-data) 이 글이 유익하니 읽어보시면 좋습니다.

ksqlDB에 대해서는 이미 이런식으로 국내에도 글들이 꽤 있어서요.

이 글에서는 중요한 특징 2개만 간단히 짚어보고, 실제 **사용 사례 위주**로 소개해드리고자 합니다.

<br/>

### SQL 기반 스트리밍 처리

이름을 보고 알 수 있듯이, SQL을 기반으로 스트리밍 처리를 할 수 있습니다.

다만 일반적인 SQL과는 조금 다르게 Push / Pull 쿼리와 스트림, 테이블 개념들이 들어가있기 때문에, 조금 더 복잡하긴합니다.

<br/>

### 카프카 단일 종속성

가장 중요한 장점입니다. 

다른 스트리밍 아키텍쳐를 설계할 때는 여러 서비스들에 대해 종속성을 가지게 됩니다.

- AWS의 경우 상황에 맞게 EMR, Kinesis, Lambda, DMS 등 사용사례에 맞게 다양한 컴포넌트를 적절히 배치 필요
- 스파크 스트리밍의 경우에도, 스파크 클러스터 내에서 여러 리소스들의 구성와 별도 스토리지 구성이 필요

한 것으로 알고 있습니다.

<br/>

여러 서비스 종속성이 늘어나면 비용이 늘어나는 것은 물론이고, 서비스 장애 시 문제를 빠르게 파악하기도, 모니터링하기도 어려워집니다.

<br/>

사실 ksqlDB도 내부적으로 RocksDB를 사용해서 데이터를 저장하는데요.

빠른 처리를 위해서 In-Memory에 들고있는 개념으로 실제 데이터는 카프카에 저장되어있습니다.

다시 ksqlDB를 띄우더라도 카프카에서 데이터를 가져와서 리플레이하기에 멱등성이 보장됩니다.

<br/>

카프카를 스토리지로 사용하고 ksqlDB는 컴퓨팅 엔진만을 필요로 하기 때문에, 손쉽게 인프라의 단순성을 확보할 수 있습니다. 

<br/>

---

<br/>

![core](/assets/img/ksqldb/core.svg "core")

이렇게 인프라의 단순성을 유지하면서도, SQL 기반의 높은 생산성을 가져갈 수 있기 때문에
- 스트림 처리 파이프라인 도입 초반에 적절할 것
- 중요한 비즈니스 로직을 구현하는데 집중할 수 있을 것

이라고 판단해서 도입하게 되었습니다.

## 배경 및 환경

- EKS
- MSK Clsuter

Confluent Cloud를 사용하는 경우 [Stream Lineage](https://docs.confluent.io/cloud/current/stream-governance/stream-lineage.html) 와 같은 형태로 데이터 흐름을 파악할 수 있고, 다른 부가 서비스도 사용할 수 있습니다.

하지만, 저는 사내에서 이미 대부분 인프라로 AWS 매니지드 서비스나 EKS를 사용하고 있어 이렇게 사용하게 되었습니다.

또한, Debezium 기반의 데이터베이스의 변경 내역만을 데이터 소스로 사용합니다.

<br/>

<br/>

## ksqlDB 구성 설정

필요한 것은 오직 카프카 클러스터와 ksqlDB 서버가 돌아갈 컴퓨팅 리소스 단 두가지입니다.

다만, 사용상의 편의를 위해서 스키마 레지스트리도 구성해두면 편리합니다.

자세한 구성에 대한 내용은 [[쿠버네티스 환경에서 ksqlDB 구성]] 글을 참고해주세요.

<br/>

## ksqlDB 사용 사례

앞선 구성 설정들을 통해서 ksqlDB 와 스키마 레지스트리를 구성해두었습니다.

이제부터 말씀드릴 3가지 사용사례의 목표는 **Braze로 적재할 사용자 속성 / 이벤트 데이터 생성** 입니다.

사용자 ID인 user_id 를 key로 갖는 어떤 사용자 행동 데이터 토픽을 생성하는 것 입니다.

<div class="mermaid">
graph LR
    A[Source Connector]
    G[Sink Connector]
    
    subgraph "이벤트 참여 토픽"
        direction LR
        B["user_id: 2<br/>D 이벤트 참여<br/>10:00:20"] --> C["user_id: 3<br/>A 이벤트 참여<br/>10:00:15"]
        C --> D["user_id: 1<br/>C 이벤트 참여<br/>10:00:10"]
        D --> E["user_id: 2<br/>B 이벤트 참여<br/>10:00:05"]
        E --> F["user_id: 1<br/>A 이벤트 참여<br/>10:00:00"]
    end
    
    A --> B
    F --> G
</div>

이 토픽은 Sink Connector를 통해서 외부 시스템인 Braze로 전송됩니다.

<br/>

### 이벤트 참여 신청 데이터 - Materialized View 와 조인 사용하기

가장 먼저, 단순한 조인 쿼리를 실행해서 가져오는 파이프라인입니다.

event 데이터베이스의 event 스키마 하위에 2가지 테이블이 있습니다.

- event.submit
- event.name

event.submit 테이블은 이벤트 참여 신청 데이터를 저장하고, event.name 테이블은 이벤트 이름 데이터를 저장합니다.

<div class="mermaid">
graph TB
    subgraph "Data Example"
        direction TB
        C[user_id: 1<br/>event_id: 101] -->|has| D[event_id: 101<br/>name: 봄맞이 이벤트]
        E[user_id: 2<br/>event_id: 101] -->|has| D
        F[user_id: 3<br/>event_id: 102] -->|has| G[event_id: 102<br/>name: 한여름 이벤트]
    end

    subgraph "ERD"
        direction TB
        A[**event.submit**<br/>
user_id: int
event_id: int] -->|has| B[**event.name**<br/>
event_id: int
name: string]
    end
    
</div>

이 두 테이블의 데이터를 조인해서 사용자 행동 데이터를 생성하고, 이벤트 토픽을 생성해 봅시다.

사실 사례는 그냥 데이터베이스에서 바로 SQL 로 짠다면 굉장히 간단한 쿼리가 됩니다. 외래키 조건에 맞게 조인만 잠깐하면 되는데요.

```sql
select
    s.user_id,
    e.name,
    s.created_at
from event.submit s
join event.name e
    on s.event_id = e.id;
```

하지만, 조인 조건이나 스키마가 복잡해질수록 쿼리가 복잡해지고, 쿼리 빈도가 높아질수록 DB 인스턴스에 더 많은 부하를 주게 됩니다.

CDC 와 ksqlDB를 조합하면 DB 인스턴스에 부하를 주지 않고 실시간으로 데이터를 처리할 수 있습니다.

<br/>

이번 사례에서는

- 테이블로, event.name 테이블의 materialized view 를 ksqlDB 에서 생성하기
- 스트림으로, event.submit 테이블에 조인해서 이벤트 참여 토픽 생성

우선 데이터 소스로 사용할 토픽을 기반으로 스트림을 생성합니다.

```sql
create stream `stg_submit`
with (
    kafka_topic='EVENT_MYSQL.EVENT.SUBMIT',
    value_format='AVRO'
);

create stream `stg_name`
with (
    kafka_topic='EVENT_MYSQL.EVENT.NAME',
    value_format='AVRO'
);
```

우선, 각 스트림의 이름을 백틱 (`)으로 감싸주었습니다.

ksql은 테이블, 스트림, 컬럼 등 각종 리소스의 이름을 대문자로 변환합니다.

따라서 이름을 백틱으로 감싸주어야 쿼리 작성 시 소문자로 지정할 수 있습니다.

저는 운영 초기이다보니 직접 cli로 접근하는 경우가 많아 가독성을 위해 백틱으로 감싸주었습니다.

<br/>

두번째로 AVRO 포맷을 사용합니다.

이는 카프카 토픽에 저장된 Debezium 데이터가 스키마 레지스트리를 통해 AVRO 포맷으로 저장되어 있기 때문입니다.

스키마 레지스트리를 사용하지 않는경우 JSON 포맷으로 저장되어, 아래와 같이 일일히 타입을 지정해줘야합니다.

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

이처럼 데베지움 메시지 포맷이 엄청 길기 때문에, 가능한 스키마 레지스트리를 사용하는 것을 권장합니다.

<br/>

스트림의 데이터를 ksql-cli를 통해서 바로 확인해보면 아래와 같습니다.

```bash
ksql> show streams;
 Table Name                      | Kafka Topic                               | Key Format | Value Format | Windowed 
--------------------------------------------------------------------------------------------------------------------
 stg_submit                      | EVENT_MYSQL.EVENT.SUBMIT                  | AVRO       | AVRO         | false    
 stg_name                        | EVENT_MYSQL.EVENT.NAME                    | AVRO       | AVRO         | false    
--------------------------------------------------------------------------------------------------------------------

ksql> select * from `stg_submit` emit changes;
+-------------------+-------------------+-------------------+------+-------------------+-------------------+
|BEFORE             |AFTER              |SOURCE             | .... |OP                 |TS_MS              |
+-------------------+-------------------+-------------------+------+-------------------+-------------------+
|null               |{user_id=1,        |{VERSION=3.0.8.Fina|      |c                  |1710921600000      |
|                   |event_id=101,      |l, TS_MS=1710921600|      |                   |                   |
|                   |created_at=2024-03-|000, ...}          |      |                   |                   |
|                   |20T10:00:00Z}      |                   |      |                   |                   |
+-------------------+-------------------+-------------------+------+-------------------+-------------------+
|null               |{user_id=2,        |{VERSION=3.0.8.Fina|      |c                  |1710921605000      |
|                   |event_id=101,      |l, TS_MS=1710921605|      |                   |                   |
|                   |created_at=2024-03-|000, ...}          |      |                   |                   |
|                   |20T10:00:05Z}      |                   |      |                   |                   |
+-------------------+-------------------+-------------------+------+-------------------+-------------------+
|null               |{user_id=3,        |{VERSION=3.0.8.Fina|      |c                  |1710921610000      |
|                   |event_id=102,      |l, TS_MS=1710921610|      |                   |                   |
|                   |created_at=2024-03-|000, ...}          |      |                   |                   |
|                   |20T10:00:10Z}      |                   |      |                   |                   |
+-------------------+-------------------+-------------------+------+-------------------+-------------------+
```

데베지움 토픽을 처음 보신 분들을 위해 설명드리자면 이렇습니다.

- OP 열에는 트랜잭션의 타입이 저장됩니다. `c`, `r`, `d`, `u` 는 각각 create, read, delete, update 를 의미합니다.
- BEFORE 열에는 트랜잭션 이전의 행 정보가 저장되어있습니다. `d`, `u` 타입의 경우에만 값이 표기됩니다.
- AFTER 열에는 트랜잭션 이후의 행 정보가 저장되어있습니다. `c`, `r`, `u` 타입의 경우에만 값이 표기됩니다.
- SOURCE 열에는 데이터 소스와 데베지움에 대한 여러 메타데이터가 저장됩니다.

이렇게 트랜잭션들이 모든 이벤트로 기록되기 때문에, `stg_name` 스트림에서 행마다 `c`, `r`, `u` 이벤트가 여럿 있다면 `event_id` 가 여러 개가 될 수 있겠죠!

바로 조인에 쓰지 못한다는 것을 불보듯 뻔한 사실입니다.

<br/>

따라서, 이벤트 이름 테이블을 생성할 때는 `event_id`를 하나로 줄이기 위해 가장 최근 데이터로 저장하는 집계 함수를 사용합니다.

```sql
create table `tb_event_name`
with (
    kafka_topic='TABLE.EVENT.NAME',
    key_format='AVRO',
    value_format='AVRO',
    partitions=1
) as
select
    AFTER->ID as `id`,
    latest_by_offset(AFTER->NAME) as `name`
from `stg_name`
where
    AFTER->ID is not null
group by AFTER->ID
emit changes;
```

- 타임윈도우 없이 집계 함수를 사용하면, 일반적으로 스트림이 아닌 테이블로 저장해야합니다.
- `latest_by_offset`은 카프카 토픽에서 오프셋을 기준으로 가장 최근 데이터를 가져오는 집계 함수입니다.
- `AFTER->ID is not null` 조건을 통해서 데이터가 있는 경우에만 테이블에 데이터를 저장합니다.

이렇게 쿼리를 작성하면, 이벤트 이름 데이터를 저장하는 테이블이 생성됩니다. 테이블은 K/V 스토어 개념입니다.

ksql-cli를 통해서 테이블을 확인해보면 아래와 같습니다.

```bash
ksql> show tables;
 Table Name                      | Kafka Topic                               | Key Format | Value Format | Windowed 
--------------------------------------------------------------------------------------------------------------------
 tb_event_name                   | TABLE.EVENT.NAME                          | AVRO       | AVRO         | false    
--------------------------------------------------------------------------------------------------------------------

ksql> select * from `tb_event_name`;
+---------+---------------------------------------+
|event_id |name                                   |
+---------+---------------------------------------+
|101      |봄 맞이 이벤트                            |
|102      |한여름 이벤트                             |
+---------+---------------------------------------+
```

이 테이블은 원본 테이블과 동일하게 쿼리 가능하지만, 원본 테이블의 실제 데이터에는 의존적이지 않은 Materialized View 의 성격을 가지고 있습니다.

정규화가 잘 되어있고, 디멘젼 테이블의 구분이 잘 되어있다면 유용하게 쓸 수 있는 패턴입니다.

이제 이벤트 이름 테이블을 생성했으니, 이제 이벤트 참여 신청 데이터와 이벤트 이름 테이블을 조인해서 이벤트 토픽을 생성해봅시다.

```sql
create stream `stm_event_name_join`
with (
    kafka_topic='STREAM.EVENT.JOIN',
    key_format='JSON',
    value_format='JSON',
    partitions=1
) as
select
    s.after->user_id as `user_id`,
    e.`name` as `event_name`,
    s.after->created_at as `event_time`,
from `stg_submit` s
left join `tb_event_name` e
    on s.after->event_id = e.`id`
where
    (s.op = 'r' or s.op = 'c')
    and s.after->event_id is not null
    and e.`id` is not null
partition by s.after->user_id
emit changes;
```

이 쿼리는 위에서 쿼리해 봤던 `stg_submit` 스트림과 `tb_event_name` 테이블을 조인해서 이벤트 스트림을 생성합니다.

쿼리를 각 부분을 설명드리면 이렇습니다.
- `r`, `c` 필터로 `read`, `create` 이벤트만 통과시킵니다.
- Materialized View 용도로 만든 테이블 `tb_event_name` 을 조인해서 `name` 컬럼을 가져왔습니다.
- 스트림 처리이다보니 스트림, 테이블 양쪽에 데이터가 없는경우에 null로 메시지가 들어기도 합니다. not null 조건들을 추가했습니다.
- partition by 절을 통해 `user_id` 를 카프카 키로 사용했습니다.

토픽으로 설정한 `STREAM.EVENT.JOIN` 토픽을 확인하면 아래와 같이 구성 되어있습니다.

```json
// Key: "1", Timestamp: 2024-03-20 10:00:00
{
    "event_name": "봄 맞이 이벤트",
    "event_time": "2024-03-20T10:00:00Z"
}

// Key: "2", Timestamp: 2024-03-20 10:00:05
{
    "event_name": "봄 맞이 이벤트",
    "event_time": "2024-03-20T10:00:05Z"
}

// Key: "3", Timestamp: 2024-03-20 10:00:10
{
    "event_name": "한여름 이벤트",
    "event_time": "2024-03-20T10:00:10Z"
}
```

- partition by 를 별도로 지정하지 않으면, 조인 조건으로 사용한 `event_id`가 자동으로 키로 설정됩니다.
- 이벤트 스트림의 성격에 따라서 파티션 불균형이 생길수 있기 떄문에, 적절한 키를 설정해야합니다.

<br/>

### 생애 최초 구매 - 

많은 비즈니스 사례들은, 유저 행동의 퍼널 상 전환률을 끌어올리기 위해 노력합니다.

![first event](/assets/img/ksqldb/first-event.jpeg "first event")

고객 획득을 위해 노력하는 퍼포먼스 마케팅 분야에서는 오래전부터 "첫 구매 혜택" 과 같은 이벤트들을 많이 진행하는데요.

ksqlDB를 통해서 첫 구매 




<br/>

### 유저별 옵션 토글 이벤트 스트림

세번째 사례는, 
