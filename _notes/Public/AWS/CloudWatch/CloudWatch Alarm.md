---
title: CloudWatch Alarm
feed: show
date: 23-10-2023
permalink: /cloudwatch-alarm
---
CloudWatch Alarm 기능을 사용하면 [[CloudWatch]] 메트릭을 감시하여 아래 작업을 수행하게 구성할 수 있습니다.

- SNS 토픽에 알림을 보내기 (주로 슬랙 알림에 사용)
- EC2 Auto Scaling
- 등등

자주 사용하면 CloudWatch Alarm이나 SNS 토픽같은 AWS 리소스가 계속 생성되기도 하고, 당연하지만 이렇게 생성되는 리소스들에 대한 IAM 관리가 조금 불편합니다.

그래서, Grafana가 구성되어있다면 이쪽에서 구성하는게 관리 상 편합니다.