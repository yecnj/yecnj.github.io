---
title: BigQuery에서 자주 쓰는 쿼리
feed: show
date: 24-10-2023
permalink: /bigquery-favorite-queries
---
BigQuery 웨어하우스 운영에 쓸만한 쿼리들을 모아둔 문서입니다.

## point-in-time 쿼리

```
select *
from `project`.`dataset`.`table`
for system_time as of timestamp_sub(current_timestamp(), interval 6 hour)
```

위 쿼리로 테이블의 특정 시점을 쿼리할 수 있습니다.

조회 가능한 기본값은 7일이고, 테이블을 복구해야하는 장애상황에 유용하게 사용할 수 있습니다.


## 테이블 ddl 가져오기

```sql
select
    table_name,
    ddl
from
 `project`.`dataset`.INFORMATION_SCHEMA.TABLES
```

## 재현 가능한 랜덤 생성하기

```sql
select 
  row_number() over() as id,
  case 
    when mod(abs(farm_fingerprint(cast(row_number() over() as string))), 100) < 50 then 1 
    else 0 
  end as class_id
from unnest(generate_array(1, 100)) as numbers
```

farm_fingerprint : 
- 파라티터를 기준으로 재현 가능한 랜덤 값을 출력합니다.
- abs 및 mod 100를 씌워서 비율처럼 사용 가능합니다.


## 쿼리 로그 확인

```sql
with raw as (
  SELECT 
    datetime(timestamp, 'Asia/Seoul') ts, 
    protopayload_auditlog.authenticationInfo.principalEmail email, 
    protopayload_auditlog.servicedata_v1_bigquery.jobCompletedEvent.job.jobConfiguration.query.query,
    ifnull(protopayload_auditlog.servicedata_v1_bigquery.jobCompletedEvent.job.jobStatistics.totalBilledBytes/1024/1024/1024, 0) billed_gb,
  FROM `project`.`dataset`.`cloudaudit_googleapis_com_data_access_*`
  where true
    and _table_suffix > '20241201'
    and protopayload_auditlog.servicedata_v1_bigquery.jobCompletedEvent.job.jobConfiguration.query.query is not null
)

select
  *
from raw 
where true
  and email = 'contact@yeonjun.kim'
order by ts desc
```

[[Cloud Logging]] 에서 설정할 수 있는 빅쿼리 쿼리 로그 테이블의 쿼리 방법 입니다.

중간부터 기본 설정이 바뀌어서 _table_suffix 대신 파티션 컬럼이 생성되는 것으로 알고 있어 일부 수정이 필요할 겁니다.