# 쿠버네티스에서 Kafka 브로커를 보호하는 PodDisruptionBudget

쿠버네티스에서 Kafka를 운영하면 노드 드레인, 커널 업데이트, DaemonSet 롤링 업데이트처럼 **의도된 노드 작업**이 잦습니다. 이때 스케줄러는 대상 노드의 Pod를 순차로 축출(evict)하는데, 아무 제약이 없으면 Kafka 브로커가 여러 개 동시에 내려가 클러스터가 불안정해집니다. 이러한 상황을 막는 쿠버네티스 표준 도구가 **PodDisruptionBudget(PDB)**입니다.

## 1. 쿠버네티스 관점의 PDB 동작

1. **Eviction API 차단기**  
   `kubectl drain`, 자동 노드 리페어, GKE/AKS 업그레이드 등은 Kubernetes Eviction API를 호출해 Pod 삭제를 요청합니다. PDB는 Eviction 요청을 intercept하여, `minAvailable` 또는 `maxUnavailable` 조건을 위반하는 요청은 kube-apiserver 단계에서 거절합니다. 따라서 노드 작업이 시작되기 전에 “얼마나 많은 Pod가 빠져도 되는지”를 PDB로 정의해야 합니다.

2. **Pod 레벨 보호**  
   StatefulSet이나 Strimzi Operator가 생성한 브로커 Pod는 모두 동일한 라벨(예: `app=kafka`)을 공유합니다. PDB는 라벨 셀렉터를 기준으로 그룹화된 Pod 집합에 대해 보호 정책을 적용하므로, 브로커·컨트롤러·ZooKeeper 등 구성 요소마다 각각 다른 PDB를 둘 수 있습니다.

3. **클러스터 이벤트와 연동**  
   PDB 상태는 `Allowed disruptions` 값으로 노출됩니다. 값이 0이면 현재 Pod 수가 `minAvailable`과 같다는 뜻이며, 이때 추가 Eviction을 시도하면 apiserver가 즉시 거절합니다. 운영자는 `kubectl get pdb -n <ns>`로 해당 값을 확인한 뒤 유지보수 순서를 조정할 수 있습니다.

## 2. PDB 값 설정 시 고려 요소

쿠버네티스에서는 “동시에 몇 개의 브로커를 비울 수 있는가?”를 기준으로 값을 정합니다. StatefulSet/Strimzi가 생성한 Pod 수, 다중 AZ 구성 여부, 브로커와 컨트롤러를 분리했는지 여부 등을 종합해 `minAvailable`(또는 `maxUnavailable`) 값을 선택합니다.

- `minAvailable`은 “살아 있어야 하는 Pod 수”, `maxUnavailable`은 “동시에 빠질 수 있는 Pod 수”를 뜻합니다. 두 값을 동시에 지정할 수 없으며, 운영 시 이해하기 쉬운 표현을 선택하면 됩니다.
- 쿠버네티스 오토스케일러나 클라우드 제공 노드 업그레이드 기능은 반드시 PDB를 존중합니다. 따라서 값을 너무 공격적으로 낮추면 노드가 한 번에 여러 개 비워져 Kafka가 위험해질 수 있습니다.

## 3. 쿠버네티스 매니페스트 예시

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: kafka-brokers-pdb
  namespace: eden
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: kafka
      strimzi.io/name: eden-kafka
```

- 라벨 셀렉터는 StatefulSet/Strimzi가 Pod에 부여하는 라벨과 일치해야 합니다.
- 멀티 네임스페이스 환경에서는 `namespace`를 명시해 배포합니다.
- 컨트롤러 NodePool 또는 ZooKeeper가 있다면 동일한 패턴으로 별도 PDB를 작성합니다.

## 4. 운영 중 확인해야 할 쿠버네티스 시나리오

1. **노드 드레인**  
   `kubectl drain <node>` 실행 시 PDB 위반으로 실패한다면, 다른 노드부터 순서를 조정하거나 브로커를 수동으로 줄인 후 재시도합니다. 메시지는 “cannot evict pod as it would violate the pod's disruption budget” 형태로 표시됩니다.

2. **클러스터 업그레이드/노드 자동 교체**  
   EKS/GKE/AKS의 노드 업그레이드는 내부적으로 Eviction API를 사용합니다. PDB가 없으면 업그레이드 시점에 여러 브로커가 동시에 내려가므로, 업그레이드 전에 `kubectl get pdb`로 `Allowed disruptions`가 0보다 큰지 확인합니다.

3. **Strimzi 롤링 업데이트**  
   Strimzi Operator가 Kafka 버전을 올리거나 설정을 변경하는 중에도 PDB를 준수합니다. PDB가 너무 빡빡하면 Strimzi 롤링이 오래 걸릴 수 있으므로, 작업 전후로 `kubectl get pdb`와 Strimzi 상태(`kubectl describe kafka <name>`)를 함께 확인합니다.

4. **장기 Pending 상태**  
   브로커 Pod가 Pending 상태에 머물러 replica 수가 줄어들면 PDB `minAvailable` 조건을 만족하지 못해 다른 Pod 축출이 차단됩니다. 이 경우 노드 자원을 확보하거나 스케줄링 제약(taint/affinity)을 조정해 정상 Pod 수를 회복해야 합니다.

## 5. 배포 패턴

쿠버네티스에서는 다음 방법으로 PDB를 관리할 수 있습니다.

1. **독립 매니페스트**  
   `kubectl apply -f pdb.yaml`로 별도 파일을 배포합니다. Strimzi CR과 분리되어 있어도 문제 없습니다.

2. **Kustomize/Helm 포함**  
   `kafka/overlays/dev` 같은 오버레이에 PDB 매니페스트를 포함하거나 Helm 차트에 추가해 환경별로 값만 덮어씁니다.

3. **GitOps**  
   Argo CD나 Flux를 사용한다면 Kafka 리소스와 동일한 Application 내에서 PDB를 선언합니다. PDB는 리소스 수가 적고 독립적이어서 GitOps 관리에 적합합니다.

쿠버네티스는 PDB를 자동으로 정리하지 않으므로, 브로커 수가 바뀌면 `minAvailable` 값도 반드시 업데이트해야 합니다. `kubectl edit pdb`나 CRD 편집이 아닌 정식 매니페스트 수정을 통해 일관성을 유지하세요.
