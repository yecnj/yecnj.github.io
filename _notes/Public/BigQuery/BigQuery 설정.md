빅쿼리에서 쓸만한 설정

external table from gcs

- storage 서비스에서 external 테이블 생성 가능

Partition

- 아시는대로 date로만 거시면 됩니다.
- 파티션당 용량이 소량인 경우, 파티션을 굳이 걸 필요 없습니다.
- require_partition_filter 거시면 반드시 파티션 컬럼을 where 절 에 넣어야 쿼리 가능

Cluster

- 카디널리티가 높지만, 연속적인 값이 아니라 이산적인 값에 사용하면 됩니다.
- 자주 where 절에 넣는 경우 효과적입니다.
- 예를 들어 고객이 잦은 빈도로 발생시키는 트랜잭션이 기록되는 테이블의 고객 id 같은 컬럼.
    - 요즘은 bigquery에서 cluster 거는곳 추천해줘서 추천대로 사용하셔도 됩니다.
        
- cluster로 용량이 절감될 수는 있지만 쿼리 실행전 콘솔에 나오면 예상 쿼리 처리량에는 적용되지 않습니다. 실제로 쿼리가 작동된 이후에 billed 된 쿼리를 확인해야합니다.
    

quotas 설정

- 조인 잘못하거나하면 작은 테이블이라도 비용을 과하게 사용할수 있음.
- quotas를 설정해두는 것이 좋습니다.

audit log 설정

- cloud logging

사용 쿼리 모니터링하기

- information_schema에서 사용 쿼리 용량 조회가능
```
SELECT
	project_id,
	user_email,
	query,
	CAST(ifnull(total_bytes_processed/1024/1024/1024, 0) AS INT64) billed_gb,
	creation_time
FROM
`region-asia-northeast3`.INFORMATION_SCHEMA.JOBS
WHERE
	creation_time BETWEEN TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 DAY) AND CURRENT_TIMESTAMP()
	AND total_bytes_processed >= 1000000000 -- 1GB
ORDER BY total_bytes_processed DESC
LIMIT 10
```
- audit 로그에서 마찬가지로 사용 쿼리 용량 조회 가능