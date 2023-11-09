---
title: Software Architecture
feed: show
date: 09-11-2023
permalink: /software-architecture
---
**소프트웨어 아키텍쳐란?**

아키텍쳐란, 소프트웨어 시스템에 대한 개발자들의 공통적인 이해사항으로, 이해 관계자, 조직, 환경, 배경과 경험 등 많은 것들의 영향을 받는다.
- 아키텍쳐는 시스템을 이해하기 위해 필요한 "구조"들의 집합이다.
- "구조"는 아래와 같은 것들로 이루어진다.
	- 소프트웨어 요소
	- 요소들간의 관계
	- 요소와 관계에 대한 속성
- 아키텍쳐를 결정하기 위해서는 아키텍쳐 결정에 중요한 요소인 아키텍쳐 드라이버(Architectural drivers)를 고려해야한다.

&nbsp;

**아키텍쳐 드라이버(Architectural drivers)**

- 기능적 요구사항 (Functional Requirements)
- 기술적 / 비즈니스적 제약사항 (Constraint)
- **품질 속성 (Quality Attributes)**

&nbsp;

**품질 속성 (Quality Attributes)**

품질 속성이란 비 기능적 요구사항이라고 불리지만, 아키텍쳐 결정에 필수적이다.
- 가용성, 변경 용이성, 성능, 보안, 테스트 용이성, 사용성과 같은 것들.
- 품질 속성은 정량적 / 독립적으로 정의되어야한다.
- 품질 속성은 재현성(reproducibility) / 반복성(repeatability)을 만족해야한다.

&nbsp;

**품질 속성 시나리오 (Quality Attributes Scenario)**

품질 속성은 기능 요구사항과 달리 정형화되지 않기때문에, 시나리오로 정리하고 아래 6개로 이루어진 공통 양식을 사용한다. 예시는 가용성 시나리오에 대한 것들이다.

- 자극 유발원(source) : 사람, 하드웨어, 소프트웨어 등
- 자극(stimulus) : 생략, 중단, 부정확한 응답 등
- 대상체(artifact) : 프로세서, 프로세스 등
- 환경(environment) : 정상 운영 중, 성능 저하 운영 중, 에러 모드 등
- 응답(response) : 결함 감지, 결함 복구 등
- 응답 측정(response measure) : 가용시간, 결함 감지 시간 등

응답 측정은 하나여야하고, 응답 측정이 여러개가 되는 경우에는 별도의 QA Scenario로 분리되어야 한다.

&nbsp;

**설계 전술(Tactics)**

QA 시나리오에서 자극에 대한 응답을 통제하기 위해 사용하고, QA 시나리오를 만들기 위한 블록이다.
- 가용성 택틱으로는 Ping/Echo, Heartbeat 와 같은 전술들이 있다.
- 변경 용이성 택틱으로는 바인딩 시점 같은 전술들이 있다.

&nbsp;

**관점 (perspective)**

사람에 대한 아키텍쳐를 볼때 외관, X-ray, MRI와 같이 여러 관점에서 볼 수 있는 것처럼, 소프트웨어 아키텍쳐를 정의하기 위해서는 3가지 관점이 필요하다.

- Static perspective
	- 모듈, 클래스와 같은 코드기반의 요소들과 그 관계
	- 변경 용이성, 테스트 용이성과 같은 품질 속성을 확인하기 위한 관점
- Dynamic perspective
	- CPU 사용량 등 런타임의 구성요소와 그 관계
	- 가용성과 같은 품질 속성을 확인하기 위한 관점
- Physical perspective
	- 서버, 센서, 입출력 장치와 같은 물리적 구성 요소들과 그 관계
	- 사용성과 같은 품질 속성을 확인하기 위한 관점

&nbsp;

이러한 각 관점에 맞춰 Module View, Component & Connector View, Allocation View 와 같이 "구조"를 설명하기 위한 뷰들이 정해져있다.

&nbsp;

**모듈 뷰 (Module View)**

Static perspective 의 뷰 타입으로, 소스 코드를 위한 청사진을 제공한다.

&nbsp;

모듈 뷰의 구조
- 요소 : 기능의 크기에 따라 나뉜다
- 관계
	- is-part-of : 부모와 자식 같은 관계, Module Level
	- depends-on : 가져다 쓰고있는 관계. 프론트엔드 모듈이 백엔드 모듈을 사용한다.
	- is-a : part-of보다 넓은 관계? lion is animal 같은 관계

- 표기법 : 텍스트, 그래픽 표기법, UML, ERM

&nbsp;

표기 스타일
- 분할 스타일 (decomposition style)
- 사용 스타일 (uses style) : depends-on 관계가 특화됨
- 레이어 스타일 (layered style) : allowed-to-use 관계

&nbsp;

**C&C 뷰 (Componets & Connector View)**

Dynamic perspective 의 뷰 타입

- 요소
	- 컴포넌트 (연산요소나 데이터 저장공간)
	- 커넥터 (런타임의 컴포넌트 간의 경로)
- 관계 : 무엇이 어디에 Attachment 되어있는지

&nbsp;

표기 스타일
- 파이프 & 필터 스타일 (pipe filter style)
- Shared data style : Blackborad / Repository 
- 발행 구독 스타일 (Pub sub style) : 생산 / 소비의 분리로 변경 용이성 향상을 위해 사용
- 클라이언트 & 서버 스타일
- peer-to-peer 스타일

&nbsp;

**할당 뷰 (Allocation View)**

Physical perspective의 뷰 타입

&nbsp;

**레퍼런스** 

책 : [소프트웨어 아키텍쳐 이론과 실제 4/e](http://www.acornpub.co.kr/book/swarchitect-4e)
