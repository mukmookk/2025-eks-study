# Kafka 네트워크 구성 핵심 개념

## 1. Headless Service가 필요한 이유

Kafka는 토픽 데이터를 파티션으로 분산 저장하고 리더–팔로워 구조로 복제합니다. 팔로워는 리더 브로커에 **직접 TCP 연결**을 맺어 데이터를 가져가므로, 모든 브로커는 재시작 이후에도 **항상 동일한 호스트 이름**으로 서로를 참조해야 합니다. 일반 Pod는 재시작 시 IP가 변경되기 때문에 리더·팔로워 간 통신 경로가 달라져 복제 지연이나 ISR 축소 같은 문제가 발생할 수 있습니다.

이를 해결하기 위해 Kubernetes에서는 **StatefulSet + Headless Service** 조합을 사용합니다. StatefulSet은 broker-0, broker-1처럼 고정된 Pod 이름을 보장하고, Headless Service는 ClusterIP 없이 Pod 이름 기반 DNS를 생성해 브로커별 고정 주소를 제공합니다.

| 브로커 | 고정 DNS 이름 |
| --- | --- |
| broker-0 | broker-0.kafka-brokers.default.svc.cluster.local |
| broker-1 | broker-1.kafka-brokers.default.svc.cluster.local |
| broker-2 | broker-2.kafka-brokers.default.svc.cluster.local |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-brokers
spec:
  clusterIP: None
  selector:
    app: kafka
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
spec:
  serviceName: kafka-brokers
  replicas: 3
  selector:
    matchLabels:
      app: kafka
```

이 구성 덕분에 브로커는 재시작 여부와 관계없이 같은 DNS 이름으로 서로를 찾을 수 있습니다.

## 2. Kubernetes 기반 Kafka 네트워크 아키텍처

Kafka는 클라이언트가 각 브로커에 **직접 연결**하는 구조이므로, Kubernetes에서도 브로커별 고정 주소와 외부 엔드포인트가 필수입니다.

### 2.1 브로커 디스커버리는 클라이언트 책임

Kafka 클라이언트는 다음 두 단계를 거칩니다.

1. 부트스트랩 서버로 초기 접속합니다.
2. 해당 브로커에서 클러스터 메타데이터를 수신합니다.

메타데이터에는 브로커 ID와 advertised 주소, 토픽·파티션 구성, 파티션 리더 정보가 포함되어 있습니다. 클라이언트는 관심 있는 파티션의 리더 주소를 확인하고, 해당 브로커에 **새로운 TCP 연결**을 열어 데이터를 주고받습니다. Kafka 브로커는 클라이언트 요청을 프록시하지 않으므로, advertised.listeners에 기록된 주소가 실제로 접근 가능한지 여부는 클라이언트 통신 성공 여부에 직결됩니다.

### 2.2 단일 LoadBalancer가 Kafka에 부적합한 이유

일반 L4 LoadBalancer/ClusterIP는 들어온 연결을 라운드로빈 혹은 임의로 Pod에 분배합니다. 그러나 Kafka는 특정 파티션 리더 브로커에 정확히 연결해야 하므로, 단일 로드밸런서 뒤에 모든 브로커를 숨기면 요청이 잘못된 브로커로 전달되어 통신이 실패합니다. 따라서 Kafka는 **브로커별 개별 엔드포인트**가 필요하며, 이를 위해 Kubernetes에서는 다음 구성을 갖춥니다.

1. **Headless Service**로 브로커 간 고정 DNS를 제공
2. **외부 부트스트랩 + 브로커별 Service**로 외부 접근 엔드포인트를 제공

Strimzi Kafka Operator는 브로커 수(N)에 맞춰 N+1개의 서비스를 자동으로 생성하고 advertised 주소를 일치시켜 이 구조를 단순화합니다.
