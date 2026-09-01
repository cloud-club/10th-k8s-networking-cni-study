# Week 4. Primary CNI

Primary CNI의 대표 사례를 중심으로 실제 동작 방식과 트래픽 흐름을 비교하는 주차입니다. Overlay와 Underlay, Routing 방식, NetworkPolicy 지원 여부를 함께 살펴봅니다.

## 학습 포인트

- Flannel Architecture / VXLAN
- Calico Architecture / Routing / BGP
- Overlay vs Underlay
- Pod-to-Pod Packet Flow 비교
- NetworkPolicy / IPAM

## 정리할 내용

- Flannel과 Calico의 네트워크 구성 방식
- 같은 Pod 통신도 CNI에 따라 어떻게 달라지는지
- 정책 제어와 IP 할당이 어떤 계층에서 처리되는지
