---
title: ksqlDB를 통한 스트리밍 파이프라인 구성
og_image: /assets/img/ksqldb/logo.png
feed: show
date: 16-05-2025
permalink: /ksqlDB
---

![ksqldb](/assets/img/ksqldb/logo.png "ksqldb")

<br/>

## 글에 들어가기 앞서
이런 개념에 대해 알고 읽으시면 좋습니다.
- SQL 사용한 데이터처리
- 큐, 카프카의 토픽, offset의 개념
- kafka 상의 파티션이 어떻게 작동하는지 간단한 개념
- 데이터 소스로 사용할 데베지움 혹은 CDC

<br/>

아래는 이 글에서 원활한 설명을 위해서 다루지 않는 내용입니다.
- 카프카 상의 파티션과 ksqlDB 상에서 사용하는 파티션에 대한 깊은 개념
- 타임 윈도우 기반의 스트림 처리
- emit final / changes 옵션에 대한 설명

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

<br/>

또한, Debezium 기반의 데이터베이스의 변경 내역만을 데이터 소스로 사용합니다.

<br/>

## ksqlDB 구성 설정

필요한 것은 오직 카프카 클러스터와 ksqlDB 서버가 돌아갈 컴퓨팅 리소스 단 두 가지입니다.

다만, 사용상의 편의를 위해서 스키마 레지스트리도 구성해두면 편리합니다.

자세한 구성에 대한 내용은 [[쿠버네티스 환경에서 ksqlDB 구성]] 글을 참고해주세요.

<br/>

## ksqlDB 사용 사례

앞선 구성 설정들을 통해서 ksqlDB 와 스키마 레지스트리를 구성해두었습니다.

이제부터 말씀드릴 3가지 사용사례의 목표는 **사용자 이벤트 데이터 생성** 입니다.

사용자 ID인 `user_id` 를 key로 갖는 어떤 사용자 행동 이벤트 토픽을 생성하는 것 입니다.

<br/>

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

이 토픽은 Sink Connector를 통해서 외부 시스템으로 전송됩니다.

<br/>

이제부터 총 3가지 사례를 알려드릴텐데요.
- 사례 1: **이벤트 참여 신청** - Materialized View 와 조인 사용하기
- 사례 2: **생애 최초 주문** - Backfill과 동시성 문제 해결하기
- 사례 3: **토글 옵션 설정** - 테이블을 활용해서 중복 제거하기


이 사례들을 통해 ksqlDB의 기본적인 기능들과 스트림 처리 패턴들에 대해서 살펴보겠습니다.

<br/>

### 사례 1: 이벤트 참여 신청 - Materialized View 와 조인 사용하기

가장 먼저, 단순한 조인 쿼리를 실행해서 가져오는 파이프라인입니다.

<br/>

event 데이터베이스의 event 스키마 하위에 2가지 테이블이 있습니다.

- `event.submit`
- `event.name`

`event.submit` 테이블은 이벤트 참여 신청 데이터를 저장하고, `event.name` 테이블은 이벤트 이름 데이터를 저장합니다.

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

두 테이블의 데이터로 `user_id` 별로 어떤 `event_name`에 참여했는지 카프카 토픽을 생성해 봅시다.

사실 이 사례는 그냥 데이터베이스에서 바로 SQL 로 짠다면 굉장히 간단한 쿼리가 됩니다. 외래키 조건에 맞게 조인만 잠깐하면 되는데요.

```sql
select
    s.user_id,
    e.name,
    s.created_at
from event.submit s
left join event.name e
    on s.event_id = e.event_id;
```

하지만, 운영 DB 인스턴스에서 조인 조건이나 스키마가 늘어날수록 쿼리가 복잡해질 수 있습니다.

또한, 쿼리 빈도가 높아져야하는 실시간 요구사항이라면 DB 인스턴스에 더 많은 부하를 주게 됩니다.

CDC와 스트림 처리를 조합하면 DB 인스턴스의 부하를 최소한으로 하되, 실시간으로 데이터를 처리할 수 있습니다.

<br/>

이번 사례에서는

- 테이블로, `event.name` 테이블의 materialized view 를 ksqlDB 에서 생성하기
- 스트림으로, `event.submit` 테이블에 조인해서 이벤트 참여 토픽 생성

에 대한 내용을 진행해보겠습니다.

우선, 데이터 소스로 사용할 토픽을 기반으로 스트림을 생성해야 스트림을 쿼리할 수 있게 됩니다.

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

첫 번째로, 각 스트림의 이름을 백틱 (`)으로 감싸주었습니다.

ksql은 테이블, 스트림, 컬럼 등 각종 리소스의 이름을 대문자로 변환합니다.

따라서 이름을 백틱으로 감싸주어야 쿼리 작성 시 소문자로 지정할 수 있습니다.

저는 운영 초기이다보니 직접 cli로 접근하는 경우가 많아 가독성을 위해 백틱으로 감싸주었습니다.

<br/>

두 번째로, AVRO 포맷을 사용합니다.

이는 카프카 토픽에 저장된 Debezium 데이터가 스키마 레지스트리를 통해 AVRO 포맷으로 저장되어 있기 때문입니다.

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
앞서 말씀드렸듯, KSQL은 모든 열을 대문자로 처리하기 때문에, 대문자로 표기되었습니다.

데베지움 토픽을 처음 보신 분들을 위해 설명드리자면 이렇습니다.
- OP 열에는 트랜잭션의 타입이 저장됩니다. `c`, `r`, `d`, `u` 는 각각 create, read, delete, update 를 의미합니다.
- BEFORE 열에는 트랜잭션 이전의 행 정보가 저장되어있습니다. `d`, `u` 타입의 경우에만 값이 표기됩니다.
- AFTER 열에는 트랜잭션 이후의 행 정보가 저장되어있습니다. `c`, `r`, `u` 타입의 경우에만 값이 표기됩니다.
- SOURCE 열에는 데이터 소스와 데베지움에 대한 여러 메타데이터가 저장됩니다.

<br/>
이렇게 트랜잭션들이 일일히 이벤트로 기록되기 때문에, `stg_name` 스트림에서 행마다 `c`, `r`, `u` 이벤트가 여럿 있을 수 있습니다.

당연히 `event_id` 가 여러 개가 될 수 있겠죠!

일반적인 DB에 쓰는 것처럼, 바로 조인에 쓰지 못할 겁니다.

<br/>

따라서, 이벤트 이름 테이블을 생성할 때는 `event_id`를 하나로 줄이기 위해 가장 최근 데이터로 저장하는 집계함수를 사용합니다.

```sql
create table `tb_event_name`
with (
    kafka_topic='TABLE.EVENT.NAME',
    key_format='AVRO',
    value_format='AVRO',
    partitions=1
) as
select
    after->id as event_id,
    latest_by_offset(after->name) as name
from `stg_name`
where
    after->id is not null
group by after->id
emit changes;
```

- latest_by_offset은 카프카 토픽에서 오프셋을 기준으로 가장 최근 데이터를 가져오는 집계 함수입니다.
- 타임윈도우 없이 집계 함수를 사용하면, 일반적으로 스트림이 아닌 테이블로 저장해야합니다.
- `after->id is not null` 조건을 통해서 데이터가 있는 경우에만 테이블에 데이터를 저장합니다.
- `after`, `id` 등을 소문자로 표기한 이유는, 백틱을 감싸지 않으면 알아서 대문자로 변환하고 쿼리하기 때문입니다.

이렇게 쿼리를 작성하면, 이벤트 이름 데이터를 저장하는 테이블이 생성됩니다. 테이블은 group by 조건에 맞게 K/V 스토어 개념이 적용됩니다.

<br/>

ksql-cli를 통해서 테이블을 확인해보면 아래와 같습니다.

```bash
ksql> show tables;
 Table Name                      | Kafka Topic                               | Key Format | Value Format | Windowed 
--------------------------------------------------------------------------------------------------------------------
 tb_event_name                   | TABLE.EVENT.NAME                          | AVRO       | AVRO         | false    
--------------------------------------------------------------------------------------------------------------------

ksql> select * from `tb_event_name`;
+---------+---------------------------------------+
|EVENT_ID |NAME                                   |
+---------+---------------------------------------+
|101      |봄 맞이 이벤트                            |
|102      |한여름 이벤트                             |
+---------+---------------------------------------+
```

이 테이블은 원본 테이블과 동일하게 쿼리할 수 있고, 운영 데이터 스토어에는 의존적이지 않은 Materialized View 의 성격을 가지고 있습니다.

정규화가 잘 되어있다면 유용하게 쓸 수 있는 패턴이여서 꼭 기억해두시면 좋습니다.

<br/>

이제 이벤트 이름 테이블을 생성했으니, 이제 이벤트 참여 신청 "스트림"과 이벤트 이름 "테이블"을 조인해서 다시 스트림을 생성해봅시다.

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
    e.name as `event_name`,
    s.after->created_at as `event_time`,
from `stg_submit` s
left join `tb_event_name` e
    on s.after->event_id = e.event_id
where
    (s.op = 'r' or s.op = 'c')
    and s.after->event_id is not null
    and e.event_id is not null
partition by s.after->user_id
emit changes;
```

이 쿼리는 위에서 쿼리해 봤던 `stg_submit` 스트림과 `tb_event_name` 테이블을 조인해서 이벤트 스트림을 생성합니다.

쿼리를 각 부분을 설명드리면 이렇습니다.
- `r`, `c` 필터로 `read`, `create` 이벤트만 통과시킵니다.
- Materialized View 용도로 만든 테이블 `tb_event_name` 을 조인해서 `name` 컬럼을 가져옵니다.
- 스트림 처리이다보니 스트림, 테이블 양쪽에 데이터가 없는경우에 null로 메시지가 들어기도 합니다. not null 조건들을 설정합니다.
- partition by 절을 통해 `user_id` 를 넣어 카프카 키로 사용합니다.

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
앞서 말했듯, 메시지의 키가 `user_id` 로 설정되어있습니다.

이제 `STREAM.EVENT.JOIN` 토픽에 Sink Connector 를 구성하면, 이벤트 참여 신청 스트림을 외부 시스템으로 전송할 수 있습니다.

---

몇가지 추가 참고사항입니다.

- partition by 를 별도로 지정하지 않으면, 조인 조건으로 사용한 `event_id`가 자동으로 키로 설정됩니다.
- 이벤트 스트림의 성격에 따라서 파티션 불균형이 생길수 있기 떄문에, 적절한 키를 설정해야합니다.
- 조인 시 스트림-스트림 / 스트림-테이블 등 관계에 따라서 동작이 달라집니다. [Confluent 문서 예시](https://docs.confluent.io/platform/current/ksqldb/developer-guide/joins/join-streams-and-tables.html)를 읽어보고, 사례에 맞게 잘 사용해주세요.

<br/>

### 사례 2: 생애 최초 주문 - Backfill과 동시성 문제 해결하기

많은 비즈니스 사례들은, 유저 행동의 퍼널 상 전환률을 끌어올리기 위해 노력합니다.

고객 획득을 위해 노력하는 퍼포먼스 마케팅 분야에서는 오래전부터 "첫 구매 혜택" 과 같은 이벤트들을 많이 진행하는데요.

![first event](/assets/img/ksqldb/first-event.jpeg "first event")

두 번째 사례는, 생애 최초 주문 여부를 판단하는 파이프라인입니다.

우선 앞서 했던것과 마찬가지로 데이터 소스로 사용할 토픽을 기반 스트림을 생성합니다.

```sql
create stream `stg_order`
with (
    kafka_topic='TRADE_MYSQL.TRADE.ORDER',
    value_format='AVRO'
);
```

이번에 사용할 `trade.order` 테이블은 주문 데이터를 저장하는 테이블입니다.

주문 데이터는 생성, 취소, 완료 등 여러 이벤트로 구성되어있습니다.

주문이 생성되면 테이블에 Insert 이벤트가 발생하고, 취소나 완료의 사유로 만료되면 Delete 이벤트가 발생합니다.

<div class="mermaid">
graph TB
    subgraph "Data Example"
        direction TB
        A[주문 생성<br/>order_id: 101<br/>user_id: 1] -->|Insert| B[**trade.order**<br/>order_id: int<br/>user_id: int]
        B -->|Delete| C[주문 취소<br/>order_id: 101]
        D[주문 생성<br/>order_id: 102<br/>user_id: 2] -->|Insert| B
        B -->|Delete| E[주문 완료<br/>order_id: 102]
    end
</div>

`order_id`는 원본 테이블인 `trade.order` 에서
- pk 컬럼이기 때문에 중복되는 값이 없고
- auto increment 컬럼이기 때문에 최초 이벤트를 판단하는데 유용합니다.

따라서, 저는 `min(order_id) group by user_id` 를 통해서 최초 이벤트를 판단하기로 했습니다.

우선, 앞서 했던것과 유사하게 재료로 쓸 "주문 생성" 이벤트만 발라내야합니다.

```sql
create stream `stm_order_candidate`
(
  user_id bigint key,
  order_id bigint
)
with (
  kafka_topic='STREAM.ORDER.CANDIDATE',
  key_format='AVRO',
  value_format='AVRO',
  partitions=1
);
```
- CSAS (CREATE STREAM AS SELECT) 이 아닌, CREATE STREAM 만을 사용해서, 빈 껍데기만 정의해줍니다.
- 따라서, 옵션으로 설정한 `STREAM.ORDER.CANDIDATE` 토픽에는 아직 아무 데이터도 들어오지 않습니다.
- 차후 group by 조건에 사용할 `user_id` 열의 타입은 `bigint key` 처럼 적어서 카프카 키로 설정해줍니다.

여기에 주문 생성 이벤트들이 있다고 생각하면서, `min(order_id)`를 구해봅시다.

```sql
create table `tb_min_order`
with (
  kafka_topic='TABLE.MIN_ORDER',
  key_format='AVRO',
  value_format='AVRO',
  partitions=1
) as
select
    user_id,
    min(order_id) as min_order_id
from `stm_order_candidate`
group by user_id;
```

`stm_order_candidate`에도 데이터를 넣지 않았기 때문에, 이 테이블에도 마찬가지로 아무 데이터도 없습니다.

<br/>

이렇게 작업한 이유는, 최초 이벤트를 판단하는 조건을 충족하는 데이터가 CDC 상에 없을 수도 있기 때문입니다.

MySQL에서 CDC에 사용하는 binlog 옵션을 활성화하는 경우, **binlog retention** 이라는 설정이 있습니다.

모든 트랜잭션 로그를 영구히 보관할 수 없으니, 일정 기간 이후 삭제되는 옵션인데요.

이 설정은 일반적으로 7일같은 수일 내의 값으로만 설정해서 아래와 같은 문제가 발생할 수 있습니다.

<br/>

<div class="mermaid">
flowchart LR
    A[**2025-01-01**<br/>order_id: 97<br/>user_id: 1]:::deleted
    B[**2025-01-09**<br/>빈로그 상에서 삭제됨]
    C1[**2025-03-01**<br/>CDC 실행 시점]:::cdc
    C[**2025-03-05**<br/>order_id: 101<br/>user_id: 1]:::new
    D[**2025-03-05**<br/>최초 주문으로 잘못 판단]

    A -.->|CDC로 전송되지 않음| C
    A -->|retention 기간 경과| B
    B --> C1
    C1 -->|CDC 시작| C
    C -->|카프카 상 최초 주문 <br/>order_id: 101| D

    classDef deleted fill:#FFE0B2,stroke:#ffb300
    classDef new fill:#C8E6C9,stroke:#388e3c
</div>

<br/>

CDC 실행시점에 이미 만료된 주문 데이터는 얻을 수 없는 문제입니다.

이 문제를 회피하기 위해서 데이터를 밀어넣기 전에, 최초 데이터 Backfill 을 우선 진행해야 합니다.

`stm_order_candidate` 스트림과 `tb_min_order` 테이블에 데이터를 넣지 않은 이유가 바로 이것입니다.

<br/>

초기 데이터 Backfill을 위해서, OLAP로 사용중인 빅쿼리에서 `min(order_id)` 를 구해서 넣어줍니다.

`stm_order_candidate` 스트림이나 `tb_min_order` 테이블에 REST API를 통해서 직접 INSERT INTO 구문을 실행합니다.

```python
query = f"""
select
    user_id,
    min(order_id) as min_order_id
from `bigquery_dataset.log_order`
group by 1
"""

print("BigQuery에서 데이터 조회 중...")
query_job = client.query(query)

# ...

# KSQL API 호출
response = requests.post(
    f"{KSQL_URL}/ksql",
    headers={
        "Content-Type": "application/json",
        "Accept": "application/json"
    },
    json={
        "ksql": f"""
            insert into `stm_order_candidate`
            (user_id, order_id)
            values ({user_id}, {min_order_id});
        """
    }
)
```

<div class="mermaid">
flowchart LR
    A[**2025-03-01**<br/>user_id: 1<br/>min_order_id: 97]:::backfill
    B[**stm_order_candidate**<br/>user_id: 1<br/>order_id: 97]:::stream
    C[**stm_order_candidate**<br/>user_id: 1<br/>order_id: 101]:::stream
    D[**2025-03-05**<br/>order_id: 101<br/>user_id: 1]:::new
    E[**tb_min_order**<br/>user_id: 1<br/>min_order_id: 97]:::table

    A -- backfill from bigquery --> B
    D -- CDC --> C
    B -- group by --> E
    C -- group by --> E

    classDef backfill fill:#E3F2FD,stroke:#1976D2
    classDef new fill:#C8E6C9,stroke:#388e3c
    classDef stream fill:#F5F5F5,stroke:#616161
    classDef table fill:#F5F5F5,stroke:#1976D2
</div>

이렇게 미리 데이터를 넣어두면, binlog 기간과 관계없이 모든 최초 주문 ID를 얻을 수 있습니다.

<br/>

Backfill이 완료되면, INSERT INTO SELECT 구문을 사용해서 실시간으로 들어오는 최초 주문 ID를 집계합니다.

```sql
insert into `stm_order_candidate`
select
    after->user_id as user_id,
    after->order_id as order_id
from `stg_order`
where op = 'c'
  and after->user_id is not null
  and after->order_id is not null
partition by after->user_id
emit changes;
```

- INSERT INTO SELECT 구문을 통해서 실시간 데이터를 집계합니다.
- 주문 생성 이벤트인 OP = 'c' 인 데이터만 추가합니다.
- 이전과 마찬가지로 여러 not null 조건들을 가지고 있습니다.
- 파티션 절을 통해서 user_id 를 키로 사용합니다.

동시에 `tb_min_order` 테이블에도 데이터가 들어오기 시작하는데요.

테이블이 완성되었다고 생각하고, 이제 최초 주문 여부를 판단할 차례입니다.

```sql
create stream `stm_min_order`
with (
  kafka_topic='STREAM.MIN_ORDER',
  key_format='JSON',
  value_format='JSON',
  partitions=1
) as 
select
    s.user_id as user_id,
    true as min_order
from `stm_order_candidate` s
left join `tb_min_order` t
  on s.user_id = t.user_id
where (t.user_id is null or t.min_order_id = s.order_id)
emit changes;
```

- 앞서 했던 이벤트 참여 신청 예시와 유사하게, 스트림-테이블 조인을 사용합니다.
- 조인 조건으로 사용한 `user_id` 는 테이블의 키로 설정되어야 합니다.

또 설명이 더 필요한 부분이 있는데요. where 절에서 두가지 조건을 사용했습니다.

이렇게 설정한 이유는 처리 순서에 따라 동시성 문제가 발생하기 때문입니다.

<br/>

<div class="mermaid">
flowchart LR
    %% 전체 플로우
    S[**stm_order_candidate**] -.->|집계| T[**tb_min_order**]
    S -.->|조인| R[**stm_min_order**]
    T -.->|조인| R

    classDef stream fill:#F5F5F5,stroke:#616161
    classDef table fill:#F5F5F5,stroke:#1976D2
    classDef result fill:#C8E6C9,stroke:#388e3c
</div>

<br/>

이렇게 설정을 하면, 조인과 집계의 순서에 따라서 결과가 달라질 수 있습니다.

- `tb_min_order` 테이블이 먼저 처리(집계)되어, 최초 주문으로 식별된 경우
- `stm_min_order` 스트림이 먼저 처리(조인)되어, 테이블에 아직 데이터가 들어오지 않은 경우

<br/>

<div class="mermaid">
flowchart LR
    subgraph "스트림이 먼저 처리"
        direction LR
        A1[**stm_order_candidate**<br/>user_id: 2<br/>order_id: 102]:::stream
        B1[**tb_min_order**<br/>user_id: null<br/>min_order_id: null]:::table
        C1[**stm_min_order**<br/>user_id: 2<br/>first_order: true]:::result

        A1 -- join user_id --> B1
        B1 -- t.user_id is null --> C1
    end

    subgraph "테이블이 먼저 처리"
        direction LR
        A2[**stm_order_candidate**<br/>user_id: 2<br/>order_id: 102]:::stream
        B2[**tb_min_order**<br/>user_id: 2<br/>min_order_id: 102]:::table
        C2[**stm_min_order**<br/>user_id: 2<br/>first_order: true]:::result

        A2 -- join user_id --> B2
        B2 -- min_order_id = order_id --> C2
    end

    classDef stream fill:#F5F5F5,stroke:#616161
    classDef table fill:#F5F5F5,stroke:#1976D2
    classDef result fill:#C8E6C9,stroke:#388e3c
</div>

테이블이 먼저 처리되면, 최초 주문으로 식별할 수 있습니다.

평범하게 `t.min_order_id = s.order_id` 조건을 사용하면, 최초 주문으로 식별할 수 있게 됩니다.

<br/>

다만, 스트림이 먼저 처리되는 경우에는 최초 주문으로 식별할 수 없습니다.

신규 유저가 발생하면 테이블에 아직 해당 유저의 데이터가 없기 때문에, left join 시에 null 값을 조회하게 됩니다.

따라서, `t.user_id is null` 조건을 통해서 최초 주문인 경우로 판단하게 했습니다.

<br/>

이렇게 처리하면 테이블이 현저히 늦게 처리되거나, 연속으로 같은 `user_id`의 최초 이벤트가 발생하는 경우 중복 처리가 발생할 수 있습니다.

그렇기 때문에, sink connector 측에서 멱등성을 가질 수 있게 구성하는 것이 안전합니다.

<br/>

추가 사항으로 주문 이벤트는 굉장히 잦은 빈도로 발생하기 때문에, 테이블이 완성되는데까지 시간이 걸렸습니다.

저는 binlog retention 3일 기준 1억건 이상의 메시지가 스트림에 쌓여있었습니다.

따라서, 저는 `tb_min_order` 의 consumer lag이 충분히 떨어지는 것을 확인한 뒤에 `stm_min_order`를 배포하였습니다.

<br/>


### 사례 3: 토글 옵션 설정 - 테이블을 활용해서 중복 제거하기

마지막 사례는, 토글 옵션 데이터를 이벤트로 만드는 방법입니다.

![toggle](/assets/img/ksqldb/toggles.jpg "toggle")

사용자가 이러한 토글 버튼을 클릭할때마다, 데이터베이스의 어딘가에 변경사항이 발생하겠죠.

예시로 사용할 데이터는 아래와 같습니다.

```sql
create stream `stg_option`
with (
    kafka_topic='SETTING_MONGO.SETTING.OPTION',
    value_format='AVRO'
);
```

setting.option 컬렉션은 MongoDB에 저장되어 있는 옵션 토글 데이터를 저장하는 컬렉션입니다.

토글 옵션 데이터는, MySQL 데이터와 마찬가지로 데베지움을 통해서 카프카 토픽에 아래와 같은 형태로 저장됩니다.

```json
// Key: {"id":"{\"$oid\": \"38vhxu2ffa...\"}"}
{
    "after": "{\"user_id\": \"{\"$numberLong\": \"1\"}\", \"toggle\": false, \"toggle2\": false}",
    "op": "u",
    ...
}

// Key: {"id":"{\"$oid\": \"38vhxu2ffa...\"}"}
{
    "after": "{\"user_id\": \"{\"$numberLong\": \"1\"}\", \"toggle\": true, \"toggle2\": false}",
    "op": "c",
    ...
}

// Key: {"id":"{\"$oid\": \"6e9x7x6toja...\"}"}
{
    "after": "{\"user_id\": \"{\"$numberLong\": \"1\"}\", \"toggle\": true, \"toggle2\": true}",
    "op": "u",
    ...
}
```

- 최초 계정 설정 시 Insert, 옵션 변경 이벤트가 발생시 Update, 회원 탈퇴 시 Delete 이벤트가 발생합니다.
- `after` 필드는 MySQL과 다르게 MongoDB의 특성상 JSON 형태 string 으로 저장됩니다.
- 하나의 행에는 `user_id` 열도 있고, 여러 개의 옵션 열 `toggle`, `toggle2` 등이 있습니다.

<br/>

앞서 했던 예시와 유사하게, 필요로 하는 이벤트를 추출해서 candidate 스트림을 생성합니다.

```sql
create stream `stm_toggle_candidate`
with (
    kafka_topic='STREAM.TOGGLE.CANDIDATE',
    key_format='AVRO',
    value_format='AVRO',
    partitions=1
) as
select
    cast(regexp_replace(extractjsonfield(AFTER, '$.user_id'), '[^0-9]', '') as bigint) as user_id,
    cast(extractjsonfield(AFTER, '$.toggle') as boolean) as toggle,
    source->ts_ms as updated_at
from `stg_option`
where
    after is not null
    and (op = 'u' or op = 'c')
    and extractjsonfield(AFTER, '$.toggle') is not null;
```

- 실제 데이터가 string 형태로 저장되어 있어, extractjsonfield 함수를 사용해서 JSON 형태로 변환하고 필요한 데이터를 가져올 수 있습니다.
- `"{"$numberLong": "1"}"` 처럼 user_id가 문자열 형태로 저장되어 있어, regexp_replace 함수를 사용해서 숫자 형태로 변환할 수 있습니다.
- 최초 옵션 설정 / 토글 이벤트만을 가져오기 위해 `op = 'u' or op = 'c'` 조건을 추가합니다.
- `toggle2`나 다른 옵션은 빼고 필요로 하는 옵션인 `toggle` 열 하나만 가져옵니다.

이렇게 한뒤, 스트림에 들어오는 데이터를 살펴보면 아래와 같습니다.


```bash
ksql> select * from `stm_toggle_candidate` emit changes;
+---------+------------------------+--------------------------------+
|user_id  |toggle                  |updated_at                      |
+---------+------------------------+--------------------------------+
|1        |false                   |1747120515000                   |
|1        |false                   |1747120567000                   |
|1        |true                    |1747120590000                   |
|1        |true                    |1747120600000                   |
|2        |false                   |1747123415000                   |
|2        |true                    |1747123416000                   |
|2        |false                   |1747123417000                   |
|3        |false                   |1747123420000                   |
+---------+------------------------+--------------------------------+
```

순서대로 들어오는 데이터를 살펴보면, 어째 같은 user_id에 대해서 같은 값의 toggle 이벤트가 연속으로 발생하고 있습니다.

<br/>

사실 당연한 이야기인데요.

toggle2 및 다른 옵션들도 있기 때문에 실제로 toggle 이벤트가 아니라도 메시지가 발생하기 때문입니다.

저희 앱 내에는 십수개의 토글 버튼이 있는데, 이 버튼들이 모두 같은 컬렉션에 이벤트를 발생시키기 때문에 더 많은 데이터가 들어옵니다.

<br/>

여기에서 실제 toggle 이벤트만 트래킹하기 위해서는 연속된 데이터의 중복 제거가 필요합니다.

false->true 및 true->false 이벤트만 트래킹하고, false->false 혹은 true->true 는 무시해야합니다.

<br/>

이 데이터를 처리하기 위해서, 앞서 했던 예시와 유사하게 latest_by_offset 를 사용해서 집계합니다.

```sql
create table `tb_toggle_changes`
with (
    kafka_topic='TABLE.TOGGLE.CHANGES',
    key_format='AVRO',
    value_format='AVRO',
    partitions=1
) as
select
    user_id,
    latest_by_offset(toggle, 2)[1] as before_toggle,
    latest_by_offset(toggle, 2)[2] as after_toggle
from `stm_toggle_candidate`
group by user_id
emit changes;
```

- latest_by_offset 함수의 두번째 인자 N으로, offset 기준으로 최종 N개의 데이터를 Array로 가져오게 설정했습니다.
- 위 예시에서는 2개의 데이터를 가져오게 설정했습니다.
- 토픽내에서 해당하는 메시지가 1개 뿐이라면, 1개의 데이터가 들어간 Array만 반환합니다.
- Array 형태로 반환된 데이터는 인덱스가 1부터 시작하며 가장 큰 인덱스에 가장 최근 데이터가 위치합니다.

이렇게 생성된 테이블에 데이터를 살펴보면 아래와 같습니다.

```bash
ksql> select * from `tb_toggle_changes` emit changes;
+---------+-----------------+-----------------+
|USER_ID  |BEFORE_TOGGLE    |AFTER_TOGGLE     |
+---------+-----------------+-----------------+
|1        |true             |true             |
|2        |true             |false            |
|2        |false            |null             |
+---------+-----------------+-----------------+
```
- USER_ID 1은 가장 최근 true -> true 이벤트가 발생했습니다.
- USER_ID 2는 가장 최근 true -> false 이벤트가 발생했습니다.
- USER_ID 3은 최초 옵션 설정 이벤트로 false 값이 들어왔습니다.

테이블을 쿼리할때는 가장 최근의 값을 가져오기 때문에, 이전 설정 이벤트는 무시됩니다.

하지만 저는 이벤트 스트림으로 가공을 해야하기 때문에, 이 테이블의 메타데이터 토픽에서 다시 스트림을 생성했습니다.

```sql
create stream `stm_toggle_changes`
with (
    kafka_topic='TABLE.TOGGLE.CHANGES',
    key_format='AVRO',
    value_format='AVRO'
);
```

이후는 간단합니다.

```sql
create stream `stm_toggle`
with (
    kafka_topic='STREAM.TOGGLE',
    key_format='JSON',
    value_format='JSON'
)
as
select
    ROWKEY as `user_id`,
    case
        when after_toggle is null then before_toggle
        when before_toggle != after_toggle then after_toggle
    end as `toggle`
from `stm_toggle_changes`
where
    after_toggle is null
    or before_toggle != after_toggle
emit changes;
```

- 테이블에서 user_id를 group by 했기때문에 Kafka Key로 설정되었습니다. ROWKEY 라는 예약어를 통해 조회할 수 있습니다.
- `after_toggle is null` 조건은, 최초 옵션 설정 이벤트 혹은 offset 기준으로 최근 데이터가 1개뿐인 경우를 의미합니다.
- `before_toggle != after_toggle` 조건은, 최근 데이터가 2개 이상이며 toggle 옵션이 변경되었을 때를 의미합니다.
- 두 경우에 따라서, 가져오게 될 컬럼이 달라집니다.

이렇게 생성된 스트림을 쿼리해보면 아래와 같습니다.

```bash
ksql> select * from `stm_toggle` emit changes;
+---------+----------------------------------------+
|user_id  |toggle                                  |
+---------+----------------------------------------+
|1        |false                                   |
|1        |true                                    |
|2        |false                                   |
|2        |true                                    |
|2        |false                                   |
|3        |false                                   |
+---------+----------------------------------------+
```

처음 이벤트 스트림과는 다르게 더 이상 중복 변경 이벤트가 발생하지 않게 잘 구성되었습니다.

이제 이 스트림과 `STREAM.TOGGLE` 토픽을 활용해서
- 토글 버튼 최종 상태를 동기화하거나
- 변경 이벤트를 트래킹해서

외부 시스템으로 보내는 Sink Connector 를 구성할 수 있습니다.

<br/>

## 끝내며

지금까지 3가지 사례를 통해서 조인, 테이블, 스트림 처리 등 기본적인 기능과 패턴을 살펴보았습니다.

<br/>

> 첫 번째 사례에서는, Materialized View 를 생성하고 이벤트 참여 신청 토픽을 생성했습니다.

비단, ksqlDB를 사용하는 아키텍쳐에서의 이야기는 아닌데요.

쓰기 워크로드(백엔드 서비스와 운영 DB) 와 읽기 워크로드(ksqlDB)를 분리한다는 관점에서, CDC 기반 스트림 처리는 서비스 유지에 도움됩니다.

단순한 예시였지만, 책임 분리 / 관심사 분리 같은 아키텍쳐 관점에서 서비스 안정성에 기여할 수 있었습니다.

<br/>

그런 사례에 적합한 Materialized View으로 쓸 수 있는 유용한 패턴에 대해서 소개드렸구요.

ksqlDB에서의 대표적인 패턴인 스트림-테이블간 조인에 대해서도 알아보았습니다.

<br/>

> 두 번째 사례에서는, 생애 최초 주문 여부를 판단하는 파이프라인을 구성했습니다.

같은 스트림으로부터 스트림과 테이블을 모두 만드는 경우, 발생할 수 있는 동시성 문제에 대해서 알아보았습니다.

제한적으로나마 ksqlDB 상에서 어떻게 동시성 문제를 해결해보았구요.

트랜잭션이 많아 초기 데이터를 정상적으로 쌓을 수 없거나 이미 만료된 경우, 수동으로 초기 데이터를 Backfill 하는 방법도 살펴보았습니다.

<br/>

> 세 번째 사례에서는, 옵션 토글 데이터를 이벤트로 만드는 방법을 살펴보았습니다.

여기에서는 간단하게 extractjsonfield, regexp_replace 등 데이터 가공 함수를 사용해보았습니다.

또한, 테이블을 통해서 중복 이벤트를 제거하고 토글 버튼 상태 변경을 동기화하는 방법을 살펴보았습니다.

<br/>

이번 포스팅에서는 조인, 테이블, 스트림 처리 등 기본적인 기능과 이를 사용한 여러 패턴들을 살펴보았습니다.

저도 ksqlDB를 고작 1개월 정도 사용했지만, 처음 사용하는 사람들에게 도움이 되었으면 좋겠다는 마음에서 글을 작성해 보았습니다.

감사합니다.
