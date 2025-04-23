---
title: Strimzi KafkaConnect 구성 및 설정 예시
feed: hide
date: 23-04-2025
permalink: /strimzi-kafkaconnect
---

> 아직 작성 중입니다 :)

실제로 클러스터를 어떤 설정으로 띄우냐는 [Strimzi 공식 문서](https://strimzi.io/docs/operators/latest/configuring.html) 를 많이 참고해야합니다.

버전에 따라서 설정 방법이 다르기도 하니, 직접 구현하게 되신다면 용례에 맞게 잘 읽어보시면 좋을 것 같습니다.

<br/>

아래 설명들은 Strimzi 0.45.0 기준 `KafkaConnect` 리소스에 설정할 수 있는 옵션들을 기반으로 합니다.

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnect
metadata:
  name: my-connect-cluster
  annotations:
    strimzi.io/use-connector-resources: "true"
spec:
  replicas: 1
```

### 이미지 변경
우선 `.spec.image` 옵션을 통해서 커넥트 클러스터에서 사용할 이미지를 변경할 수 있습니다.

플러그인은 `/opt/kafka/plugins` 디렉토리에 두는 것이 권장사항 인데요.

커스텀 이미지 작업을 통해서 필요한 여러 플러그인들을 미리 넣어둘 수 있습니다.

<br/>

커넥터 플러그인들 뿐만 아니라
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

공식 설명에서 제공하는 Dockerfile 예시는 아래와 같습니다.

```dockerfile
FROM quay.io/strimzi/kafka:0.45.0-kafka-3.9.0
USER root:root
COPY ./my-plugins/ /opt/kafka/plugins/
USER 1001
```

### Truststore 설정

간혹 가다보면, 커넥터에서 특정 프로토콜을 사용하는 경우 특정 클라이언트 인증을 필요로 합니다.

예를 들어, Amazon DocumentDB의 [설명서](https://docs.aws.amazon.com/ko_kr/documentdb/latest/developerguide/connect_programmatically.html) 에 따르면 특정버전 이상부터 퍼블릭 키를 반드시 요구하는데요.

> 전송 중인 데이터를 암호화하려면 다음 작업을 사용하여 이름이 global-bundle.pem인 Amazon DocumentDB에 대한 퍼블릭 키를 다운로드합니다.
>
> wget https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem


<br/>

이런 경우, 커넥트 클러스터에서 사용할 수 있는 Truststore 설정을 추가해야합니다.

dockerfile 내에서 기본 truststore 설정을 복사한 다음, DocumentDB가 요구하는 퍼블릭 키를 추가하는 단계를 만들었습니다.

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



### nodeAffinity 설정


### AWS Secrets Manager 설정
사실 이건 Strimzi 관련이 아니라, 그냥 커넥트 클러스터의 config 입니다.

사내에서 AWS Secrets Manager를 사용하는 경우 `.spec.config` 설정으로 커넥터 클러스터에 시크릿 제공자 설정을 할 수 있습니다.

- 


### Prometheus 설정


### volume 및 env 설정
비교적 최근부터 추가된 기능으로, 커넥트 클러스터 내부에서 사용할 수 있는 볼륨과 환경변수를 설정할 수 있습니다.
- 


### 커넥트 클러스터 접근제어

저는 Redpanda 콘솔을 통해서 Connect 클러스터에 접근하고 싶었는데요

![connect-cluster-resource](/assets/img/kafka/connect-cluster-resource.jpg "connect-cluster-resource")

위에서 한 설정들로 커넥트 클러스터 리소스를 띄워보면 하위 리소스로 netpol이 함께 올라오며 API 포트의 접근이 막혀있습니다.

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

## 불편했던 점

- https://github.com/strimzi/strimzi-kafka-operator/issues/6339
- https://github.com/orgs/strimzi/discussions/6853

## 완성

이렇게 완성된 yaml (저는 Helm Chart였습니다) 를 배포하면 커넥트 클러스터가 잘 띄워집니다.

![connect-cluster](/assets/img/kafka/connect-cluster.jpg "connect-cluster")