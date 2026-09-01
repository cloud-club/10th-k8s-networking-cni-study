# Week 2. Kubernetes Networking & CNI Fundamentals

Kubernetes 네트워크 모델과 CNI가 연결되는 흐름을 이해하는 주차입니다. kubelet, CRI, CNI가 Pod 생성 시 어떤 순서로 동작하는지 확인하고, Service 트래픽이 Pod까지 전달되는 과정을 추적합니다.

## 학습 포인트

- Kubernetes Networking Model
- kubelet / CRI / CNI 동작 흐름
- Pod Network / Pod CIDR / IPAM
- kube-proxy와 Service
- Service → Pod Packet Flow 추적

## 정리할 내용

- Pod IP 할당 과정
- Service가 백엔드 Pod로 연결되는 방식
- CNI가 네트워크를 구성하는 책임 범위
