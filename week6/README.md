# Week 6. Multi-Networking & Secondary Networking

하나의 Pod에 여러 네트워크를 붙이는 방식과 이를 위한 CNI 체인을 다루는 주차입니다. Multus를 기준으로 primary/secondary network 구성과 특수 NIC 활용 방식을 정리합니다.

## 학습 포인트

- Multus Architecture / CNI Chaining
- Primary / Secondary Network
- NetworkAttachmentDefinition
- macvlan / ipvlan / SR-IOV
- Multi-NIC Pod 구성 및 Packet Flow

## 정리할 내용

- 멀티 네트워크가 필요한 상황
- 각 네트워크 방식이 적합한 사용 사례
- Pod 내부에서 네트워크 인터페이스가 여러 개일 때의 흐름
