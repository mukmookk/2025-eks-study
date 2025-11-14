# Kafka 브로커 Pod 분산 전략

Kafka 브로커가 동일 노드나 동일 AZ에 몰리면 장애 시 레플리카가 동시에 사라질 위험이 있으므로, Kubernetes 스케줄링 기능으로 명시적인 분산 정책을 지정해야 합니다. 대표 도구는 **Pod Anti-affinity**, **topologySpreadConstraints**, **nodeAffinity**입니다.

## 1. Pod Anti-affinity로 노드 간 분산

Pod Anti-affinity는 특정 라벨을 가진 Pod가 동일 노드에 함께 배치되지 않도록 합니다. Kafka 브로커에는 다음과 같이 적용할 수 있습니다.

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: kafka
            role: broker
        topologyKey: kubernetes.io/hostname
```

위 구성은 같은 노드에 `app=kafka, role=broker` 라벨의 파드가 이미 있다면 새 브로커 파드를 스케줄하지 않습니다.

### 1.1 특정 역할이 있는 노드 회피

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: role
              operator: In
              values: ["connect", "schemaregistry"]
        topologyKey: kubernetes.io/hostname
```

Connect나 Schema Registry가 있는 노드에 브로커가 배치되지 않도록 금지할 수 있습니다.

### 1.2 다중 토폴로지 적용

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - topologyKey: topology.kubernetes.io/zone
        labelSelector:
          matchLabels:
            app: kafka
            role: broker
      - topologyKey: kubernetes.io/hostname
        labelSelector:
          matchLabels:
            app: kafka
            role: broker
```

노드와 존 모두에서 충돌을 방지해 특정 AZ나 노드에 브로커가 몰리는 것을 막습니다.

### 1.3 Soft Constraint

노드 자원이 부족할 때 예외를 허용하되 가능한 한 분산하도록 선호 규칙을 추가할 수 있습니다.

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: kafka
              role: broker
          topologyKey: kubernetes.io/hostname
```

필요하면 Connect나 Schema Registry와의 공존을 피하는 선호 규칙도 `matchExpressions`로 추가할 수 있습니다.

## 2. TopologySpreadConstraints로 AZ 균등 분산

`topologySpreadConstraints`는 특정 라벨을 가진 Pod가 지정된 토폴로지 키(topology.kubernetes.io/zone 등)에 균등하게 배치되도록 강제합니다.

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: kafka
```

`maxSkew: 1`은 AZ 간 파드 개수 차이를 1로 제한하며, 조건을 만족하지 못하면 스케줄링을 중단합니다.

## 3. Node Affinity로 AZ 고정 배치

네트워크 디스크는 AZ 간 Attach가 불가능하므로, 브로커를 특정 존에 고정해야 할 때 `nodeAffinity`를 사용합니다.

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: topology.kubernetes.io/zone
              operator: In
              values:
                - ap-northeast-2a
```

브로커 ID별로 다른 AZ를 지정하면 데이터 복제본이 여러 존에 분산되어 단일 AZ 장애를 견딜 수 있습니다.

## 4. 스케줄러 동작 순서

스케줄러는 다음 순서로 규칙을 적용합니다.

1. **Filter 단계**: 하드 제약(`requiredDuringSchedulingIgnoredDuringExecution`, nodeAffinity, taint/toleration 등)을 만족하지 않는 노드를 제외합니다.
2. **Score 단계**: 소프트 제약(`preferredDuringSchedulingIgnoredDuringExecution`)을 점수화해 노드별 점수를 계산합니다.
3. **Normalize/Balance 단계**: `LeastAllocated`, `BalancedAllocation` 등 다른 스코어 플러그인의 점수와 함께 정규화합니다.
4. **Tie-break 단계**: 점수가 동점일 경우 노드명을 기준으로 결정하거나 랜덤 셔플로 선택합니다.
5. **Bind 단계**: 최종 노드에 파드를 바인딩합니다.

이 과정을 이해하면 하드 제약과 소프트 제약을 적절히 조합해 원하는 분산 정책을 만들 수 있습니다.

## 5. Strimzi 자동화와 커스터마이징

Strimzi는 기본적으로 브로커 간 Pod Anti-affinity와 rack awareness(`kafka.spec.kafka.broker.rack`)를 제공합니다. 그러나 특정 AZ 고정, zone 개수 대비 브로커 수가 다를 때의 균등 분산, 명시적인 `topologySpreadConstraints`가 필요하다면 CR의 `template.pod` 섹션에 직접 정의해야 합니다.
