# Kafka 영구 스토리지 구성

Kafka 브로커는 노드 이동 이후에도 같은 로그 데이터를 유지해야 하므로, **StatefulSet + PersistentVolumeClaim** 구조와 **네트워크 스토리지(EBS, GCE PD 등)** 사용이 표준입니다. 네트워크 스토리지는 노드와 독립적으로 존재하므로 장애 노드의 디스크를 다른 노드로 쉽게 Attach할 수 있습니다.

## 1. StatefulSet + PVC로 브로커별 디스크 할당

StatefulSet의 `volumeClaimTemplates`는 브로커 번호에 맞춰 고정 PVC를 만듭니다.

```yaml
volumeClaimTemplates:
  - metadata:
      name: kafka-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp2
      resources:
        requests:
          storage: 1Gi
```

위 설정을 적용하면 `kafka-data-kafka-0`, `kafka-data-kafka-1`, `kafka-data-kafka-2`와 같은 PVC가 생성되고, Pod가 다른 노드로 이동하더라도 동일 PVC를 다시 마운트합니다. 테스트 목적이라면 위처럼 `storage: 1Gi`, `storageClassName: gp2`로 시작하고, 용량이 필요해지면 값을 늘리면 됩니다.

Strimzi Kafka CR에서도 `.spec.kafka.storage`를 아래와 같이 선언하면 동일 효과를 얻을 수 있습니다.

```yaml
spec:
  kafka:
    storage:
      type: persistent-claim
      size: 1Gi
      class: gp2
      deleteClaim: false
```

이렇게 선언하면 Strimzi가 StatefulSet과 PVC를 자동 생성하며, 지정한 StorageClass(gp2) 기반으로 각 브로커별 1Gi 네트워크 디스크가 할당됩니다.

## 2. 네트워크 스토리지의 장점

1. **노드 장애 대응**: 노드 장애가 발생해도 Pod가 새 노드로 스케줄되며 동일 PVC를 Attach해 빠르게 복구합니다.
2. **데이터 무결성**: PVC 수명주기가 노드와 분리되어 확장·교체 작업 중에도 디스크를 유지할 수 있습니다.
3. **토폴로지 변경 안정성**: Autoscaling이나 AZ 재배치가 일어나도 브로커 정체성과 로그 데이터가 유지됩니다.
