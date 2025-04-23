---
title: Strimzi KafkaConnector 구성 및 설정 예시
feed: hide
date: 23-04-2025
permalink: /strimzi-kafkaconnector
---

> 아직 작성 중입니다 :)

<div class="mermaid">
flowchart TB
    subgraph "Git Repository"
        CR[KafkaConnector CR<br>MySQL Source<br>BigQuery Sink]
    end

    subgraph "Kubernetes"
        direction TB
        A0[KafkaConnector]
        SO[Strimzi Operator]
        
        subgraph "Connect Cluster"
            direction TB
            
            subgraph "Connect Pod 1"
                direction TB
                W1[Worker Process]
                subgraph "Active Connectors 1"
                    C1[MySQL Source<br>Task 1]
                    C2[BigQuery Sink<br>Task 1]
                end
            end
            
            subgraph "Connect Pod 2"
                direction TB
                W2[Worker Process]
                subgraph "Active Connectors 2"
                    C3[MySQL Source<br>Task 2]
                    C4[BigQuery Sink<br>Task 2]
                end
            end
        end
    end

    subgraph "External Systems"
        direction LR
        MySQL[(MySQL DB)]
        BQ[(BigQuery)]
    end

    CR -->|Deploy by ArgoCD| A0
    SO --> |Watch| A0
    SO -->|Deploy| W1 & W2
    C1 & C3 -->|Read| MySQL
    C2 & C4 -->|Write| BQ
    
    style CR fill:#BBDEFB,color:#333
    style SO fill:#FFE0B2,color:#333
    style W1 fill:#B2DFDB,color:#333
    style W2 fill:#B2DFDB,color:#333
    style C1 fill:#C8E6C9,color:#333
    style C3 fill:#C8E6C9,color:#333
    style C2 fill:#BBDEFB,color:#333
    style C4 fill:#BBDEFB,color:#333
    style A0 fill:#FFE0B2,color:#333
    style MySQL fill:#B2DFDB,color:#333
    style BQ fill:#BBDEFB,color:#333
</div>