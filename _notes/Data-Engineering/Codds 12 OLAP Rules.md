---
title: Codds의 12가지 OLAP 규칙
feed: hide
date: 09-01-2025
permalink: /codds-12-olap-rules
---

1993년 OLAP라는 개념이 처음 제시되었을때, Codds의 12가지 OLAP 규칙이 제시되었다.

이 규칙은 데이터 웨어하우스의 설계에 영향을 미치는 중요한 지침을 제공한다.

0. **기초 규칙 (Foundation Rule)**
   - 시스템은 관계형 기능을 통해서만 데이터베이스를 관리할 수 있어야 한다.

1. **정보 규칙 (Information Rule)**
   - 모든 정보는 테이블의 값으로 논리적 수준에서 명시적으로 표현되어야 한다.

2. **보장된 접근 규칙 (Guaranteed Access Rule)**
   - 모든 데이터는 테이블 이름, 기본 키 값, 열 이름의 조합으로 접근 가능해야 한다.

3. **NULL 값의 체계적 처리 (Systematic Treatment of Null Values)**
   - NULL 값은 누락된 데이터나 적용할 수 없는 데이터를 나타내며, 데이터 유형과 독립적으로 처리되어야 한다.

4. **동적 온라인 카탈로그 (Dynamic Online Catalog)**
   - 데이터베이스 구조는 관계형 모델에 따라 카탈로그에 저장되어야 한다.

5. **포괄적인 데이터 하위 언어 (Comprehensive Data Sublanguage)**
   - 시스템은 최소한 하나의 언어를 지원해야 하며, 이는 데이터 정의, 조작, 보안 등을 포함해야 한다.

6. **뷰 갱신 규칙 (View Updating Rule)**
   - 이론적으로 갱신 가능한 모든 뷰는 시스템에 의해 갱신 가능해야 한다.

7. **고수준 삽입, 갱신, 삭제 (High-Level Insert, Update, and Delete)**
   - 기본 및 파생된 관계에 대한 단일 작업이 데이터 검색뿐만 아니라 삽입, 갱신, 삭제에도 적용되어야 한다.

8. **물리적 데이터 독립성 (Physical Data Independence)**
   - 응용 프로그램은 데이터의 물리적 저장 방식이나 접근 방식이 변경되어도 영향을 받지 않아야 한다.

9. **논리적 데이터 독립성 (Logical Data Independence)**
   - 응용 프로그램은 테이블의 논리적 구조가 변경되어도 영향을 받지 않아야 한다.

10. **무결성 독립성 (Integrity Independence)**
    - 무결성 제약조건은 데이터베이스 카탈로그에 저장되어야 하며, 응용 프로그램과 독립적이어야 한다.

11. **분산 독립성 (Distribution Independence)**
    - 데이터베이스가 분산되어 있더라도 응용 프로그램과 데이터베이스 작업은 영향을 받지 않아야 한다.

12. **비하위 버전 규칙 (Nonsubversion Rule)**
    - 낮은 수준의 인터페이스가 있다면, 이는 높은 수준의 관계형 인터페이스의 무결성을 우회할 수 없어야 한다.

[출처: Wikipedia - Codd's 12 rules](https://en.wikipedia.org/wiki/Codd%27s_12_rules)
