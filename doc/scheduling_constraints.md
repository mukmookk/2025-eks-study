# Kafka 운영 문서 인덱스

주제별 세부 문서를 아래에서 확인할 수 있습니다.

1. `kafka/network_fundamentals.md`  
   - Headless Service 필요성, 브로커 고정 DNS, Kubernetes에서의 Kafka 네트워크 기본 구조를 설명합니다.

2. `kafka/strimzi_network_services.md`  
   - Strimzi 리스너 설정, 자동 생성되는 Service 종류, advertised.listeners 처리 방식을 다룹니다.

3. `kafka/storage_pvc.md`  
   - StatefulSet/Strimzi 기반 영구 스토리지 구성, gp2 1Gi 테스트 구성을 포함합니다.

4. `kafka/pod_distribution.md`  
   - Pod Anti-affinity, topologySpreadConstraints, nodeAffinity 등 분산 스케줄링 전략과 스케줄러 동작 순서를 정리합니다.

5. `kafka/pod_disruption_budget.md`  
   - 의도된 중단 시 브로커 가용성을 지키기 위한 PodDisruptionBudget 구성 방법을 설명합니다.

각 문서는 “입니다” 체로 작성되어 있어 Strimzi 기반 Kafka 운영 문서에 그대로 포함할 수 있습니다.
