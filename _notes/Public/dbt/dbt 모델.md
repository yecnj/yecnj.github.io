---
title: dbt 모델
feed: show
date: 10-11-2021
permalink: /dbt-model
---
dbt 프로젝트의 `models` 폴더 아래에 생성되는 `sql` 파일입니다.

- `sql` 파일의 이름이 모델의 이름이 됩니다.
- 각 모델은 `select` 문이 실행되게 작성됩니다.
- 각 모델은 `table` 혹은 `view table` 로 생성됩니다. 

이러한 모델을 만드는것을 dbt에서는 모델링이라고 부르며, dbt 작업 시간의 대부분을 모델링에 사용하게 됩니다.

각 모델의 디펜던시를 [ref 함수](https://docs.getdbt.com/reference/dbt-jinja-functions/ref)를 사용해서 미리 정의해 둘 수 있습니다.

```SQL
select * from {{ ref("stg_user") }}
```

&nbsp;

dbt에서는 만들어지는 모델의 형식을 강제하지는 않지만, 효율적으로 운영하기 위한 [가이드라인](https://docs.getdbt.com/guides/best-practices/how-we-structure/1-guide-overview#guide-structure-overview) 을 제공합니다.

&nbsp;

**Staging**

웨어하우스에 로드되어 있는 로우 데이터를 가져오는 첫번째 단계의 모델입니다.

- 허용되는 변환
	- 리네임, 타입 캐스팅, 기본 단위 통일 등
	- 분리보관으로 인한 완전한 동일한 스키마의 union
- 허용되지 않는 변환
	- join, 집계 등

일반적으로 스테이징에 로드될때 

&nbsp;

**Intermediate**

&nbsp;

3. **Marts** : 


`models` 하위의 폴더 구조는 
