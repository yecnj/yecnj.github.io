---
title: dbt
feed: show
date: 09-11-2023
permalink: /dbt
---
**[dbt(Data Build Tool)](https://www.getdbt.com/)** 
- ELT Pipeline에서 극단적으로 Transform 과정을 모두 관리하기 위한 도구.
- SQL과 jinja template만을 사용해 워크플로우를 관리한다.
	- [[dbt 모델]] 간의 디펜던시
	- snapshot 등 기능으로 자동으로 테이블 설계
- 오픈소스인 dbt-core와 클라우드 서비스로 제공하는 dbt Cloud가 있음.

&nbsp;
  
**dbt를 사용하지 않았을때 불편했던 경험들**
- Airflow에서,
	- 파악하거나 구현하기 어려운 수많은 모델(DAG)간의 의존도
	- 일반 SQL 만으로 사용하기 어려워 파이썬으로 적혀있는 로직
	- Airflow Pod에 대한 성능 의존도
- 웨어하우스에서,
	- 한번 짜두면 여기저기 텍스트로 공유되는 쿼리
	- 수동으로 생성해두고 용도를 까먹은 집계 테이블
- 위의 단점들로 인한 어려워지는 쿼리 및 데이터 관리

&nbsp;

**dbt를 사용하고 나서 얻을 수 있었던 것**
- SQL 쿼리의 가독성 / 생산성 / 형상 관리가 용이해짐.
- 업계 선두를 달리는 dbt 관련 오픈소스 생태계.
- 클라우드 웨어하우스에 대한 독립적인 성능 의존도.
- 쿼리 관리 문제가 다소 해소되었으나, 완전히 해결되지는 않음.

&nbsp;

**dbt를 사용하면서도 고민되는 것**
- dbt가 아니였으면 하지 못했을 일들 -> dbt 의존도가 높아짐
- 모델링을 맡는 직무(Analytics Engineer)의 모호함
	- 분석가의 필요에 의한 모델링
	- 그러나, incremental 로직 등 모델링에 엔지니어적 역량이 필요한 경우가 많음
	- 엔지니어 업무에 더 가까운거 같긴함.
- 그외 자잘한 것들 (docs 작성하기 불편함, 등등)

&nbsp;

**이런 기능들이 있음**
- 

**참고**
- [dbt 튜토리얼](https://docs.getdbt.com/quickstarts)
- [[dbt 디렉터리 구조]]
- [[dbt 모델]]
- [[dbt 명령어]]

