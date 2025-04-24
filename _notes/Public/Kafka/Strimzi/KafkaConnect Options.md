---
title: Strimzi KafkaConnect 구성 및 설정 예시
feed: show
date: 24-04-2025
permalink: /strimzi-kafkaconnect
---

실제로 클러스터를 어떤 설정으로 띄우냐는 [Strimzi 공식 문서](https://strimzi.io/docs/operators/latest/configuring.html) 를 많이 참고해야합니다.

버전에 따라서 설정 방법이 다르기도 하니, 직접 구현하게 되신다면 용례에 맞게 잘 읽어보시면 좋을 것 같습니다.

<br/>

### 이 문서에서는

Strimzi 0.45.0 기준 `KafkaConnect` 리소스에 설정할 수 있는 옵션들을 배경으로 설명합니다.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: my-connect-cluster
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  replicas: 1
  ...
```

이 문서에서는 아래 설정들의 용례와 예시를 제공합니다.

- 커넥트 이미지 변경
- Truststore 설정
- Affinity 설정
- EKS 환경에서 ServiceAccount 및 AWS Secrets Manager 설정
- Prometheus 설정
- volume 및 env 설정
- 커넥트 클러스터 접근제어

<br/>

마지막으로, 커넥트 클러스터를 배포하면서 불편했던 점들도 함께 설명합니다.

<br/>

## 주요 설정들
### 이미지 변경
우선 `.spec.image` 옵션을 통해서 커넥트 클러스터에서 사용할 이미지를 변경할 수 있습니다.

플러그인은 `/opt/kafka/plugins` 디렉토리에 두는 것이 권장사항 인데요.

커스텀 이미지 작업을 통해서 필요한 여러 플러그인들을 미리 넣어둘 수 있습니다.

<br/>

기본적으로 커넥터 플러그인들이 필요하구요.
- io.debezium.connector.mysql.MySqlConnector
- io.debezium.connector.postgresql.PostgresConnector

<br/>

Single Message Transform을 사용하는 경우 필요한 플러그인
- com.github.jcustenborder.kafka.connect.transform.common.ExtractTimestamp

<br/>

provider 설정에 따라서 플러그인을 추가하기도 해야합니다.
- com.github.jcustenborder.kafka.config.aws.SecretsManagerConfigProvider

<br/>


이 디렉토리에 커넥터 플러그인을 넣어두면 커넥트 클러스터가 자동으로 플러그인을 인식합니다.

저는 사내에서 따로 구현한 SMT도 있기 때문에 필요한 플러그인을 모두 빌드하고, 이미지를 빌드하는 CI 파이프라인을 만들어 두었습니다.

<div class="mermaid">
flowchart TB
    subgraph "GitHub Repository"
        direction TB
        SRC1[Maven / Confluent Connectors]
        SRC2[BigQuery Connector]
        SRC3[Custom SMT]
    end

    subgraph "GitHub Actions Pipeline"
        direction TB
        B1[Build Maven Connectors]
        B2[Build BigQuery Connector]
        B3[Build Custom SMT]
        COMBINE[Combine Artifacts]
        DOCKER[Build Docker Image]
    end

    subgraph "AWS ECR"
        IMAGE[Connect Image with all plugins]
    end

    SRC1 --> B1
    SRC2 --> B2
    SRC3 --> B3

    B1 & B2 & B3 --> COMBINE
    COMBINE --> DOCKER
    DOCKER -->|Push| IMAGE

    style SRC1 fill:#f6f8fa,stroke:#666
    style SRC2 fill:#f6f8fa,stroke:#666
    style SRC3 fill:#f6f8fa,stroke:#666
    
    style B1 fill:#B2D8E6,color:#fff
    style B2 fill:#B2D8E6,color:#fff
    style B3 fill:#B2D8E6,color:#fff
    style COMBINE fill:#B2D8E6,color:#fff
    style DOCKER fill:#B2D8E6,color:#fff
    
    style IMAGE fill:#FFE0B2,stroke:#232F3E
</div>

이를 통해서
- Maven / Confluent 와 같은 원격 저장소에서 플러그인을 받아오거나
- 직접 빌드한 플러그인을 사용할 수 있었습니다.

빌드 과정에 사용하기 위해 공식문서에서 제공하는 Dockerfile 예시는 아래와 같습니다.

```dockerfile
FROM quay.io/strimzi/kafka:0.45.0-kafka-3.9.0
USER root:root
COPY ./my-plugins/ /opt/kafka/plugins/
USER 1001
```

<br/>

### Truststore 설정

간혹 가다보면, 커넥터에서 특정 프로토콜을 사용하는 경우 특정 클라이언트 인증을 필요로 합니다.

예를 들어, Amazon DocumentDB의 [설명서](https://docs.aws.amazon.com/ko_kr/documentdb/latest/developerguide/connect_programmatically.html) 에 따르면 특정버전 이상부터 퍼블릭 키를 반드시 요구하는데요.

> 전송 중인 데이터를 암호화하려면 다음 작업을 사용하여 이름이 global-bundle.pem인 Amazon DocumentDB에 대한 퍼블릭 키를 다운로드합니다.
>
> wget https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem

<br/>

이런 경우, 커넥트 클러스터에서 사용할 수 있는 Truststore 설정을 추가해야합니다.

Dockerfile 내에서 기본 truststore 설정을 복사한 다음, DocumentDB가 요구하는 퍼블릭 키를 추가하는 단계를 만들었습니다.

기본은 Amazon DocumentDB의 가이드를 참고했고, `$JAVA_HOME/lib/security/cacerts` 의 기존 인증서 정보를 복사하는 과정만 추가했습니다.

```dockerfile
# RDS 글로벌 인증서 번들 다운로드 및 신뢰 저장소 추가 구성
RUN mkdir -p /tmp/certs && \
    cp $JAVA_HOME/lib/security/cacerts /tmp/certs/rds-truststore.jks && \
    chmod +w /tmp/certs/rds-truststore.jks && \
    curl -sS "https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem" > /tmp/certs/global-bundle.pem && \
    cd /tmp/certs && \
    awk 'split_after == 1 {n++;split_after=0} /-----END CERTIFICATE-----/ {split_after=1}{print > "rds-ca-" n ".pem"}' < global-bundle.pem && \
    for CERT in rds-ca-*; do \
      alias=$(openssl x509 -noout -subject -in "$CERT" | sed -e 's/^subject= //' -e 's/, /, /g'); \
      keytool -import -file "$CERT" -alias "$alias" -storepass changeit -keystore rds-truststore.jks -noprompt; \
      rm "$CERT"; \
    done && \
    rm global-bundle.pem && \
    mv rds-truststore.jks /opt/kafka/custom-truststore.jks
```

<br/>

### Affinity 설정

쿠버네티스에서는 nodeSelector 혹은 affinity 설정을 통해서 파드가 배치될 노드를 제한하며 이용하는 것이 일반적입니다.

`.spec.template.pod.affinity`에 nodeAffinity 관련 설정을 작성할 수 있습니다.

<br/>

### EKS 환경에서 ServiceAccount 및 AWS Secrets Manager 설정
Secrets Manager와 같은 외부 구성 공급자를 사용하는 경우, 커넥트 클러스터에서 사용할 수 있는 설정을 추가해야합니다.

사실 이건 Strimzi 관련이 아니라, 그냥 커넥트 클러스터의 config 입니다.

사내에서 AWS Secrets Manager를 사용하는 경우 `.spec.config` 설정으로 커넥터 클러스터에 시크릿 제공자 설정을 할 수 있습니다.

참고 : [MSK 튜토리얼: 구성 공급자를 사용하여 민감한 정보 외부화](https://docs.aws.amazon.com/ko_kr/msk/latest/developerguide/msk-connect-config-provider.html)

```yaml
spec:
  config:
    config.providers: "secretManager"
    config.providers.secretManager.class: "io.confluent.csid.config.provider.aws.SecretsManagerConfigProvider"
    config.providers.secretManager.param.aws.region: "ap-northeast-2"
    config.providers.secretManager.param.secret.prefix: "data_kafka_connect_"
```

이 설정을 사용하기 위해서는 KafkaConnect 파드가 사용하는 serviceAccount가 특정 Role을 가지고 있어야합니다.

해당 Role에서는 사용할 Secret Manager에 대한 해석권한과 이 Secret이 사용하는 KMS 키에 대한 권한이 필요합니다.

이 설정은 `.spec.template.serviceAccount` 에 작성할 수 있습니다.

```yaml
spec:
  template:
    serviceAccount:
      metadata:
        eks.amazonaws.com/role-arn: arn:aws:iam::<account-id>:role/<role-name>
```

추가로 EKS 클러스터에서 사용하는 IAM 역할이 metadata에 적혀있는 Role을 가져올 수 있게, [Trust Relationship 설정](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/cluster-iam-role.html)이 필요합니다.


<br/>

### Prometheus 설정
커넥트 클러스터의 모니터링 역시 굉장히 일반적인 설정입니다.

사내에서 이미 AMP(Amazon Managed Prometheus)를 사용하고 있어서, 이를 통해 모니터링 데이터를 수집하였습니다.

이 설정은 `.spec.metricsConfig` 에 작성할 수 있습니다.

```yaml
spec:
  metricsConfig:
  type: jmxPrometheusExporter
  valueFrom:
    configMapKeyRef:
      name: connect-metrics
      key: metrics-config.yml
```

이 configMap의 내용은 [Strimzi 레포](https://github.com/strimzi/strimzi-kafka-operator/blob/main/examples/metrics/kafka-connect-metrics.yaml)에서 확인 가능합니다.

이 설정은 Pod의 9404 포트로 JMX 메트릭을 Prometheus 형식으로 노출합니다.

<br/>

이후, 사내 AMP 설정에 따라서 Pod에 annotation을 추가해주면 모니터링 데이터를 수집할 수 있었습니다.

annotation 이름을 검색해보면 굉장히 일반적인 설정이라, 많은 경우에 사용할 수 있을 것 같아 작성했습니다.

`.spec.template.pod.metadata`에 아래 설정을 추가해주면 됩니다.

```yaml
pod:
  metadata:
    annotations:
      prometheus.io/port: '9404'
      prometheus.io/scrape: 'true'
```

이렇게 수집된 Prometheus 데이터를 조회하기 위한 [Grafana 대시보드 설정](https://github.com/strimzi/strimzi-kafka-operator/blob/main/examples/metrics/grafana-dashboards/strimzi-kafka-connect.json)도 함께 확인할 수 있습니다.

이를 통해서, 커넥트 클러스터 파드들의 상태, 각 커넥터 프로세스의 상태 등 주요한 메트릭들을 확인할 수 있습니다.

![connect-grafana](/assets/img/kafka/connect-grafana.jpg "connect-grafana")

<br/>

### volume 및 env 설정
비교적 최근부터 추가된 기능으로, 커넥트 클러스터 내부에서 사용할 수 있는 볼륨과 환경변수를 설정할 수 있습니다.

커넥터 설정에서 사용하기 위해 gcp의 credential json 파일 등이 필요할 때, 볼륨 마운트나 환경변수를 주입하는 과정이 요구되는데요.

`.spec.template.connectContainer` 에 필요한 볼륨 마운트와 환경변수를 정의할 수 있습니다.

```yaml
spec:
  template:
    connectContainer:
      env:
        - name: GOOGLE_APPLICATION_CREDENTIALS
          value: /mnt/service-account/gcp-credential.json
      volumeMounts:
        - mountPath: /mnt/service-account
          name: gcp-credentials
          readOnly: true
```

볼륨 마운트에 사용하는 볼륨은 `.spec.template.pod.volumes`에 정의할 수 있습니다.

```yaml
spec:
  template:
    pod:
      volumes:
        - name: gcp-credentials
          secret:
            secretName: connect-cluster-secrets
            items:
              - key: gcp-credential.json
                path: gcp-credential.json
```

<br/>

### 커넥트 클러스터 접근제어

저는 Redpanda 콘솔을 통해서 Connect 클러스터에 접근하고 싶었는데요.

위에서 한 설정들로 커넥트 클러스터 리소스를 띄워보면, 하위 리소스로 netpol이 함께 올라오며 API 포트의 접근이 막혀있습니다.

![connect-cluster-resource](/assets/img/kafka/connect-cluster-resource.jpg "connect-cluster-resource")

```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            strimzi.io/cluster: connect-cluster
            strimzi.io/kind: KafkaConnect
            strimzi.io/name: connect-cluster-connect
      - podSelector:
          matchLabels:
            strimzi.io/kind: cluster-operator
    ports:
      - port: 8083
        protocol: TCP
  - ports:
      - port: 9404
        protocol: TCP
```

- strimzi 관련 라벨을 가진 파드들만 8083 포트로 접근 가능
- 일반적으로 9404 포트로 접근 가능

하게 되어있습니다. 

Operator에 의해서 관리되는 하위 리소스이기때문에, 변경은 어렵고 추가 netpol 리소스를 띄워서 접근을 허용해주면 됩니다.

## 사실은 불편했던 점

이렇게 커넥트 클러스터를 잘 쓸 수 있게 만들었지만, 배포하는 과정에서 여럿 불편한 점들이 있었습니다.

우선, strimzi 의 커스텀 리소스가 생성하는 파드에 initContainer나 sidecar 컨테이너를 설정할 수 없어서 불편했습니다. 
- [커스텀 initContainer 지원이 가능한지](https://github.com/strimzi/strimzi-kafka-operator/issues/6339)
- [initContainer 사용자 지정 명령 실행](https://github.com/orgs/strimzi/discussions/6853)
- [Sidecar, initContainer 등의 추가 컨테이너 지원 요청](https://github.com/strimzi/strimzi-kafka-operator/issues/2976)

내부에 어떤 고충이 있는지 모르니 함부로 말할 수는 없긴 한데요.

많은 사용자들이 다양한 추가 컨테이너 사용사례를 제시함에도, 안된다고 일축하는 답변들만 있어 좀 아쉽긴 했습니다.

이번에 사용했던, 볼륨이나 환경변수 관련한 설정들도 최초에는 지원할 계획이 없었던 것 같습니다.

<br/>

또, 지원되는 볼륨 종류가 제한적이여서 아쉬웠습니다.

문서에 따르면 현재는 아래 다섯 개 볼륨 타입을 지원하는데요.
- Secret
- ConfigMap
- EmptyDir
- PersistentVolumeClaims
- CSI Volumes

GCP의 쿠버네티스 인증 방식으로 권고되는 [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation-with-kubernetes#kubernetes)를 사용하려면 projected 타입의 볼륨이 필요합니다.

이 문제는 해결하지 못해서, 구 인증방식인 serviceaccount json 파일을 통해서 인증하는 방식으로 해결했습니다.


## 완성

이렇게 완성된 yaml (저는 Helm Chart였습니다) 를 배포하면 커넥트 클러스터가 잘 띄워집니다.

![connect-cluster](/assets/img/kafka/connect-cluster.jpg "connect-cluster")