---
title: AWS MSK (Managed Streaming for Kafka)
feed: show
date: 16-04-2025
permalink: /kafka-msk
summary: MSK의 소개
---

MSK는 Managed Streaming for Kafka 의 약자로, 카프카 클러스터를 쉽게 배포할 수 있도록 도와주는 AWS 서비스입니다.

- [MSK 공식 문서](https://aws.amazon.com/ko/msk/)
- [MSK 기능](https://aws.amazon.com/ko/msk/features/)

<br/>

이미 인프라가 AWS 를 기반으로 구축되어있다면 다음과 같은 AWS 서비스들과의 통합을 통해 많은 이점을 얻을 수 있습니다:

- 이미 구성된 [[VPC]] 내에서의 보안된 통신
- [[AWS IAM]]과의 통합으로 인한 접근 제어
- [[CloudWatch]]와의 통합으로 모니터링 용이
- 웹 인터페이스 설정하는 만으로 Auto Scaling 환경을 구성할 수 있음

직접 카프카 클러스터를 구축해서 쓰는 편에 비해, 가용성 / 확장성 / 모니터링 용이성 / 보안 면에서의 다양한 이점이 있습니다.

다만, 매니지드 서비스인 만큼 카프카 클러스터를 직접 구축하는 것에 비해 제한적인 부분도 있고, 비용적인 면에서도 조금 더 비싼 편입니다.