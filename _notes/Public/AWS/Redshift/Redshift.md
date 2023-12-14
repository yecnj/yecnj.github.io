---
title: Redshift
feed: show
date: 14-12-2023
permalink: /redshift
---
Redshift에는 프로비전드와 서버리스 두가지 클러스터가 있다.
#### 프로비전드 클러스터
- 직접 클러스터를 구성해서 사용하는 프로비전드 클러스터가 있다.
- [요금](https://aws.amazon.com/ko/redshift/pricing/)에서 온디맨드 요금을 확인해보자. 
- 인스턴스 유형 (RA3, DC2)
	- 프로비전드 클러스터는 관리형 노드인 RA3의 경우 최소 단위라도 꽤 비싸다. 다른 현재 세대 컴퓨팅 노드인 DC2에 비해서 비슷한 스펙에 4배 가량 비싸다.
	- 차후에 기술하겠지만, Redshift를 사용해보면 데이터를 각 노드에 분산해서 저장하기 떄문에 RDB 만큼은 아니지만 다른 열 기반 압축 스토리지보다 용량이 좀 더 나오긴한다. 일반적으로 프로비전드 클러스터를 고려할 조직에서 볼만한 데이터 셋은 대개 1TB 이상일거라 생각한다. 또한 이 경우 DC2가 아닌 RA3 를 권장한다.
- 컴퓨팅 단위는 클러스터 ~ 노드 - 슬라이스
	- 노드가 2개인 클러스터를 생성되면 리더 노드 1개와 컴퓨팅 노드 2개가 할당된다. 리더 노드의 비용은 따로 부과되지 않는다. 다만, 노드 1개짜리 클러스터를 선택하면 리더 노드가 컴퓨팅 노드를 겸하게 된다.
- 각 노드당 기본 슬라이스 몇개가 있어서 그만큼 병렬처리가 가능하다.

&nbsp;

#### 서버리스 클러스터
- 인스턴스를 소유하지 않고, 데이터를 적재해둔 스토리지 비용만 상시로 받는다. RA3 타입의 관리형 스토리지 가격으로 책정된다. RA3 뿐만 아니라 다른 클라우드 웨어하우스 스토리지는 그리 비싸지는 않은듯. 보통 프로비전드 대신 서버리스를 사용한다.
- **요금**
	- 당연히 서버리스니까 pay per use.
	- 다만, 다른 서버리스 웨어하우스 제품과는 다르게 병렬로 쿼리 실행시에는 각각 받지않고 사용한 시간에 따라서만 과금된다.
- **Data Sharing** : 워크로드의 상황에 따라서 프로비전드 클러스터와 혼용 가능
	- 현재는 혼용시에는 타 클러스터에 대해 Read만 가능하다.
- 클러스터마다 RPU 라는 컴퓨팅 자원 할당 단위을 설정 가능
	- 빅쿼리 슬롯 같은건가 싶음.
	- 목적에 맞는 RPU를 각각 할당해놓고 쓸 수 있음
	- ex) 분석용 RPU,  인하우스용 RPU 등등

&nbsp;

#### Redshift에서 읽어볼만한 기능

- **서버리스**
- **Automatic materialized view** : 차후에 기술하는 materialized view의 자동 버전.
- **Data sharing** : Redshift 클러스터 사이에 데이터 조회를 공유하는 기능. 쿼리를 실행하는 쪽 비용을 사용한다.
- **Streaming ingestion** : 같은 AWS 서비스인 Kinesis 혹은 DMS 로부터 Redshift를 타겟으로 바로 지정 가능하다.
- **Zero ETL** : 최근 정식 릴리즈된 기능으로, Aurora -> Redshift로 자동 싱크를 관리해준다. 서울에서는 아직 미리보기인가? 현재는 Aurora MySQL 3 버전 이상에서만 지원된다. Zero ETL은 CDC를 위해서 MySQL 빈로그에 의존하기 때문에 빈로그가 켜져있어야 한다. Aurora를 사용하면서 빈로그를 켜야하는 것은 아쉬운 점.
- **RA3 node / storage** : S3를 기반으로 한다. RA3 스토리지 타입을 사용한다면 스냅샷을 찍어 프로비전드 <-> 서버리스간에 전환이 가능한 것 같다.
- **Redshift Specturm** : S3에 있는 데이터를 바로 읽을 수 있는 기능으로 같은 기능을 제공하는 AWS 서비스인 Athena (2.x - presto, 3.x - trino)와는 전혀 다른 엔진.
- **Auto copy from S3** : S3에 새 blob이 생기면 Redshift로 로드하는 COPY 문을 자동으로 실행시켜 주는 것 같다. 문제는 트랜잭션이 많은 경우 S3에서 직접 쿼리하거나 직접 COPY를 실행시키는게 별로 어렵지는 않다는 점.

&nbsp;

#### Redshift 최적화 기법

테이블 분산 스타일 DISSTYLE (KEY, ALL, EVEN) 
- KEY로 하면 각 키별로 컴퓨팅 노드의 각 슬라이스에 저장됨. 중요하다.
- ALL로 하면 모든 컴퓨팅 노드의 첫번째 슬라이스에 각각 저장됨.
- EVEN 들어오는 순서대로 슬라이스에 분산 저장됨.
- skewed 될수있기 때문에 분산 스타일을 잘 지정저장하는 것이 중요함.

&nbsp;

Redistribution
- star, snowflake schema에서 각 테이블의 분산스타일이 적합하지 않은 분산키를 선택했을때 실행

&nbsp;

network broadccast
- 쿼리 실행 플랜 평가(ds_dist_none, ds_dist_all_none, ds_dist_inner)
- 노드간에, 슬라이스간에 서로 주고받는게 많은 경우 좋지 않아지는 식의 평가들이 있음.

&nbsp;

어떻게 분산 스타일을 잡는지?
- 1순위
	- 가장 큰 디멘젼 테이블의 primary key
	- fact 테이블의 foreign키를 dis key로 선택
		- not skewed, 높은 cardinality여야 한다.
- 2순위
	- 나머지 조인이 되는 조건의 디멘젼 테이블은 dimension all을 검토.
	- 대충 300만 row 이하인 경우.
- ALL로 하면 안좋은점
	- 테이블에 업데이트가 발생하면, 모든 노드에 업데이트가 발생해서 손해.

&nbsp;

데이터셋 사이즈
- load를 할때, 1mb짜리 블럭에 저장함.
	- delete를 하면 실제로 안지우고 마킹만해둠
	- 저장해놓고 vaccum 돌리면 그때 지움.

&nbsp;

성능 올리는법
- node 확장!
- 병렬성 높이기
- 네트워크 트래픽 줄이기.

&nbsp;

결국엔 disk io 줄이기. 그 방법들은,
- 정렬키
- materialized view
- vacuum
- compress

&nbsp;

정렬키
- RDB와 같은 인덱스는 없지만, 비슷하게 대응하는 정렬키라는 기능이 있다.
- blocks
	- 1mb immutable 으로 이루어진 블록들
	- encoding type의 영향을 받는다.
- zone map
	- 1mb 블럭마다 컬럼마다 min max를 저장해둠
	- 쿼리를 날리면 그 컬럼의 값을 meta에서 검색하고 그 블럭을 가져와서 쿼리.
	- 인덱스랑 비슷하다 -> 빅쿼리 파티션이랑 조금 유사한듯.
- 정렬키를 여러개 잡는다면?
	- 카디널리티가 높은걸 앞으로 잡으면 손해 -> 순서대로 찾기 때문에 트리로 찾을수있게 낮은값부터 순서대로 잡는게 좋다. ex) 연-월-일
- interleaved sort key (정렬키) -> 쓰지않기.
	- compound 쓰는게 sharing되고 좋다.
- best practice
	- 날짜가 일반적
	- 자주 조인하면? 정렬키와 분산키를 모두 조인 컬럼으로 잡기.

&nbsp;

materialized view
- 분산스타일이나 분산키, 정렬키를 원래 테이블과 별도로 가져갈 수 있음.
- 옵티마이저가 알아서 만들어서 쓰기도함.

&nbsp;

vaccum
- 정렬키를 소팅하거나 마킹된 블록을 실제로 삭제
- full, sort only, delete only, reindex (interleaved sortkey)
- 알아서 돌리기도하고, 직접 테이블별로 선택해서 돌릴 수도 있음

&nbsp;

compress
- 컬럼별 인코딩 타입
- 정렬키에 의한 블록에 영향을 준다.

&nbsp;

Explain Query
- cost 높은거
- join type (nested loop, hash join)
- inner outer (smallere for inner )

&nbsp;

svv_query_summary
- 실제 수행된 쿼리의 정보 -> 오래 걸린 쿼리가 있으면 플랜을 확인해야한다.
- 실행 전 쿼리 플랜과는 다른게 실제로 쿼리 컴파일후에 다른 쿼리가 돌기도 한다.

&nbsp;
### Redshift hacks
- asterisk 금지
- union 대신 union all 대신 case when
- 정렬키 기준으로 필터
- similar < like < condition
- where 절 안에 함수 사용하지 않기 - 정렬키가 적용되지 않음
- orderby나 group by 에서 정렬키 순서대로면 좋음 (카디널리티가 낮은값 순서)
- subquery로 해서 join을 줄이기. -> join보다는 분산처리가 빠름.
- cross join은 피하기

&nbsp;
### Redshift 운영 모니터링 

파라미터 그룹
- 여러개 있긴한데, 중요한건 
- auto_analyze=true 테이블 통계정보
- max_concurrency_scaling_clusters=1
	- WL에서 큐할당을 하는데 더많은 쿼리가 나오면, 똑같은 클러스터가 하나 더 확장되는 식

&nbsp;

WLM (Workload Management)
- 쿼리 등을 식별하고 각 큐에 라우팅해서 리소스를 관리하는 기능
- 사용자 그룹 / 쿼리 그룹 지정 가능
- 수동 WLM
	- 사용자가 직접 워크로드별 쿼리 대기열 설정
- 자동 WLM
	- Redshift가 알아서 우선순위 설정하고 순서대로 처리
- Queues (basic WLM)
- Short-Query Acceleration (SQA) : 숏 쿼리는 자동으로 큐에 넣어서 바로 실행
- 워크로드 큐 -> 수정가능
	- 해당하는 파라미터 그룹에서 대기열 편접 가능함. Auto WLM이기 때문에 수동으로 전환 후 사용. 들어오는 쿼리 성격을 모르겠다면 Auto 계속 써도됨. 특정 쿼리만 계속 온다면 수동으로해도됨.
- WLM 속성에 따라서 클러스터 재부팅이 필요할 수 있음.
- 큐마다 메모리 / 동시성 / 제한시간 같은 설정 가능.
- 슈퍼유저 대기열 / 대기열 할당 규칙 같은 설정 가능.
- Query Monitoring Rule (QMR)
	- nested join, 너무 많은 행 리턴 / 조인 같은 룰 베이스 기반

&nbsp;

모니터링
- 쿼리수
- 디스크 공간
- CPU 
- 동시성 그래프 등
- 그라파나 Redhsift 대시보드도 있음

&nbsp;

운영관리
- resize
	- classic도 아주 차이나진 않음.
- advisor
- analyze
- vacuum
	- auto vacuum은 기본은 full로 돔. 매 1시간마다 돌지만, 워크로드가 급증하면 안됨. 클라우드 워치에 auto vaccum space freed 지표를 보면 됨. 잘 안되면 어드바이저에서 리포트가 되기도 함.
	- 정렬은 95%이상 되어있으면 스킵함.