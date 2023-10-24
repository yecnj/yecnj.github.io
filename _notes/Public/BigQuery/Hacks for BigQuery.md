---
title: Hacks for BigQuery
feed: show
date: 24-10-2023
permalink: /bigquery-hacks
---
BigQuery 웨어하우스 운영에 쓸만한 기능을 모아둔 문서입니다.

## 쿼리
### point-in-time 쿼리

```
select *
from `project.dataset.table`
for system_time as of timestamp_sub(current_timestamp(), interval 6 hour)
```

위 쿼리로 테이블의 특정 시점을 쿼리할 수 있습니다.

조회 가능한 기본값은 7일이고, 테이블을 복구해야하는 장애상황에 유용하게 사용할 수 있습니다.
