---
title: Braze Currents
feed: hide
date: 06-02-2025
permalink: /braze-currents
summary: Braze Currents를 사용해서 Braze 데이터를 웨어하우스에 동기화하기
---

[Braze Currents](https://www.braze.com/docs/user_guide/data/braze_currents/) 라는 추가 유료기능을 사용하면, Braze 데이터를 외부 플랫폼에 동기화할 수 있습니다.

Currents를 통해서
- S3, Google Cloud Storage, Azure Blob Storage 같은 퍼블릭 클라우드 스토리지
- Mixpanel, Amplitude 같은 플랫폼

으로 데이터를 동기화 해줄 수 있습니다.

동기화하는 방법은 [문서](https://www.braze.com/docs/user_guide/data/braze_currents/setting_up_currents)를 참고하였습니다.

<br/>

저는 BigQuery를 웨어하우스로 사용하고 있었기 때문에 Google Cloud Storage로 동기화를 해주었습니다.

일단, Braze 콘솔의 Parteners Integration > Data Export 에서 설정이 가능합니다.

![Currents Configuration](/assets/img/braze-in-data-org/currents-config.png "Currents Configuration")

Google Cloud Storage로 동기화할때는 [[Service Account]] JSON 파일과 동기화할 버킷명을 입력해주면 됩니다.

S3 버킷으로 동기화할때는 JSON 파일 대신 Access Key ID, Secret Access Key 혹은 Role ARN 정보를 요구합니다.

각 Service Account 혹은 권한 정보는 버킷에 대한 Write 권한을 가지고 있어야합니다.

자세한 권한은 각 스토리지 연동 가이드를 확인해보면 좋습니다.

<br/>

만약 동기화가 잘 되지 않는다면 이미 사용중인 AWS, GCP 상에 있는 방화벽을 확인해서 각 braze 인스턴스 IP를 IP Allowlist에 추가해주면 됩니다.

<br/>

하단에는 어떤 이벤트들을 동기화할지 설정할 수 있으나, 대부분의 상황에서 모든 이벤트를 설정하는 것이 좋습니다.

설정을 완료하고, GCS에 동기화된 데이터들을 살펴보면 이렇게 생겼습니다.

![Currents GCS](/assets/img/braze-in-data-org/currents-gcs-folder.png "Currents GCS")

파일 트리 최하단에는 각종 유저 이벤트들이 avro 형식의 파일로 저장되어 있습니다.

이를 BigQuery 외부 테이블로 생성합니다.

```sql
create external table `<gcp-project>.<gcp-dataset>.messages_pushnotification_send`
with partition columns
options(
  format="avro",
  hive_partition_uri_prefix="gs://<bucket-name>/currents/dataexport.<braze-project-id>.<braze-region>.integration.<braze-instance-id>/event_type=users.messages.pushnotification.send",
  uris=["gs://<bucket-name>/currents/dataexport.<braze-project-id>.<braze-region>.integration.<braze-instance-id>/event_type=users.messages.pushnotification.send/*.avro"]
);
```

이제 이 외부 테이블은 다른 BigQuery 테이블과 거의 동일하기 때문에, 기존에 쓰던 도구들과 통합하여 조회가 가능합니다.
