---
title: Warehouse for Application
feed: hide
date: 21-11-2023
permalink: /warehouse-for-application
---
웨어하우스를 운영하다 보면 데이터 조직 내외부를 가리지 않고 이런 의견들이 생긴다.
- Relational DBMS에서는 못 돌리는 쿼리인데, 웨어하우스에서는 엄청 빠르네요?
- 그러니까, 서비스에서 쿼리하고 싶어요.
- 클라우드 웨어하우스(BigQuery, Snowflake 등)의 성격은 서비스에서 사용하기에 적합하지 않아요.
	- Latency : 
	- Freshness : 
- 그래도 그걸 감안하고 사용하고 싶어요.

&nbsp;

API Server가 있어야하는 이유
- Latency
- 클라우드 웨어하우스의 비용

&nbsp;

이 API Server를 
- 데이터 조직이 만드냐 / 개발 조직이 만드냐 에 대해서
- 데이터 조직은 가용성이 높은 서비스 설계 경험이 없어서 직접 만들기 어려웠다.
	- 또한, 웨어하우스의 변환 파이프라인에 서비스 유지보수 일감이 따라올까 걱정된다.
- 그냥 크레덴셜만 뽑아주자니, 잘못된 쿼리를 사용 / 유지보수 할까봐 곤란했다.

&nbsp;

고민을 하다가, 잘 만들어진 도구들이 있는지 알아봤다.

- OLAP 엔진 단에서 개선
	- [Apache Pinot](https://pinot.apache.org/)
		- 최근 (2023-09-19)에 v1.0.0 릴리즈가 되었다.
		- User Facing Analytics 라고 표현함.
- Semantic Layer
	- cube for dbt
	- dbt cloud
