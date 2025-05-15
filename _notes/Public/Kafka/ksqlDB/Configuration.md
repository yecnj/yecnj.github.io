---
title: 쿠버네티스 환경에서 ksqlDB 구성
feed: hide
date: 13-05-2025
permalink: /ksqlDB-configuration
---

> 글을 작성하는 중입니다 :)

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
