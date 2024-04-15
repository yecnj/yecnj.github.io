---
title: 앱 Uninstall 트래킹
feed: show
date: 15-04-2024
permalink: /product-uninstall-traking
---

**background**

- 분석가 분들이 이탈 관련 이야기할때 여기저기 물어 주워들은 얘기
- CRM Tool 들에서 제공하는 Uninstall 이벤트에 관련한 이슈.. 를 열심히 뜯어본 경험은 아니고 옆에서 본 감상

<br/>

**detail**

braze, airbridge 등 CRM 툴, 마테크 솔루션 등등 이렇게 불리는 물건들에서 측정하는 앱 Uninstall 이벤트는 실제로 앱을 삭제하는 이벤트를 정확히 트래킹할 수 없다. FCM, APNs를 통한 앱 푸시 이벤트 기반으로 동작한다.

기본적으로 앱 푸시 이벤트를 발생시켰을 때, 푸시를 받지 못하는 상태라면 바운스 몇번 하다가 Uninstall 이벤트로 기록하는 방식이고 자세하게는 CRM 툴에 따라서 다르지만

- 미이탈로 생각되는 사용자에게 주기적으로 사일런트 푸시를 보내서 트래킹
- 캠페인 등 푸시 발송 시점에 트래킹

이런 방법들을 혼용하는 것처럼 보였다. 이렇기 때문에 실제로 Uninstall 이벤트 관련해서 이슈가 많다.

<br/>

아래와 같은 경우 Uninstall 이벤트가 실제와 다르게 측정될 수 있다.

- 사용자의 기기에서 네트워크 이슈로 인해 앱 푸시를 수신할 수 없었던 경우
- 사용자의 기기에서 앱 데이터 삭제, 로그인 세션 등을 이유로 FCM token 이 만료되는 경우
- 여러 번 푸시를 보낸 경우 install 없이 Uninstall 이벤트가 연속으로 찍히는 경우(CRM 툴에 따라 다름)

<br/>

또, 측정 시점과 관련된 이슈들도 있다.

- 푸시를 보내야 삭제 감지가 가능하기 때문에 푸시 빈도가 낮은 경우 해상도가 낮아지고 대조군 설정이 제한될 수 있음
- 토큰 갱신 후 FCM, APNs에서 감지하는데에 n일 이상이 걸리기도 함


<br/>

푸시 토큰 갱신 등 어플리케이션 상 구현된 코드에 영향을 받기도 해서 파악하기 어려운 주제인 것 같기도하고.

애초에 왜 Uninstall이라고 부르는 것인가 싶기도하고.

직접 데이터를 모두 뜯어본 것은 아니라 확실하지는 않지만 Uninstall 이벤트를 제대로 확인하기 위해서는 푸시 메시지를 보내는 빈도, 사일런트 푸시 설정 등 각 회사에서 현재 CRM 도구를 운영하는 방식에 맞게 위양성을 제거하거나 중복 Uninstall을 제거하여 이탈을 재정의할 필요가 있지않을까 싶었다.


<br/>

**relates to**

<https://help.airbridge.io/ko/guides/uninstall-event-tracking>

<https://www.braze.com/docs/user_guide/data_and_analytics/tracking/uninstall_tracking/>