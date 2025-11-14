# Strimzi 기반 Kafka 네트워크 구성

Strimzi Kafka Operator는 `.spec.kafka.listeners`에 정의한 정보를 토대로 StatefulSet, Service, advertised.listeners를 자동으로 생성·관리합니다.

## 1. 내부·외부 리스너 설정 예시

### 1.1 LoadBalancer 기반 내부/외부 리스너

```yaml
listeners:
  - name: plain
    port: 9092
    type: internal
    tls: false
  - name: external
    port: 9094
    type: loadbalancer
    tls: true
```

- `plain` 리스너는 클러스터 내부용 PLAINTEXT 접속입니다.
- `external` 리스너는 LoadBalancer로 노출되며 TLS를 사용합니다.

### 1.2 NodePort 기반 외부 리스너

```yaml
listeners:
  - name: external2
    port: 9094
    type: nodeport
    tls: true
    authentication:
      type: tls
```

NodePort 리스너는 `externalTrafficPolicy`나 `preferredNodePortAddressType`을 통해 advertised 주소에 사용할 노드 IP 유형을 선택할 수 있습니다.

## 2. Strimzi가 생성하는 Service 유형

클러스터 이름이 `my-cluster`라고 가정하면 다음 네 가지 서비스가 생성됩니다.

1. **my-cluster-kafka-bootstrap**: 내부 부트스트랩용 ClusterIP 서비스입니다.
2. **my-cluster-kafka-brokers**: 브로커 간 통신을 위한 Headless Service입니다. 예) `my-cluster-kafka-0.my-cluster-kafka-brokers.svc`.
3. **my-cluster-kafka-<listener>-bootstrap**: 외부 부트스트랩(NodePort/LoadBalancer) 서비스입니다.
4. **my-cluster-kafka-<listener>-<broker번호>**: 브로커별 외부 서비스입니다. 예) `my-cluster-kafka-external-0`.

외부 부트스트랩 서비스는 초기 연결만 담당하며, 실제 데이터 전송은 브로커별 서비스로 이동합니다.

## 3. advertised.listeners 자동 설정

Strimzi는 Service 상태 정보를 읽어 각 브로커의 advertised.listeners를 업데이트합니다.

- **LoadBalancer**: `.status.loadBalancer.ingress[].hostname` 또는 IP를 추출해 사용합니다.
- **NodePort**: 클러스터 노드 IP와 NodePort를 조합해 `<노드IP>:<NodePort>` 형태로 기록하며, `preferredNodePortAddressType`으로 InternalIP/ExternalIP 등을 선택할 수 있습니다.

advertised 주소는 Kafka 리소스의 `.status.listeners`에 기록되며, 다음 명령으로 외부 부트스트랩 주소를 확인할 수 있습니다.

```bash
kubectl get kafka my-cluster \
  -o=jsonpath='{.status.listeners[?(@.name=="external")].bootstrapServers}'
```
