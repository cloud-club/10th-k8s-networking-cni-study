# 10th Kubernetes Networking CNI Study

Kubernetes 네트워킹과 CNI의 동작을 이해하고, 실제로 어떻게 Pod 네트워크가 구성되는지 추적하는 스터디 저장소입니다.

## 스터디 목표

- Pod에 IP가 어떻게 할당되는지 이해하기
- Pod-to-Pod, Pod-to-Service, External 통신 흐름 추적하기
- CNI가 어떤 역할을 하고 왜 필요한지 이해하기
- Flannel, Calico, Cilium, Multus, macvlan/ipvlan, SR-IOV 등 비교하기
- Linux 네트워크 기본 개념(veth, bridge, routing, netfilter, iptables, eBPF)과 Kubernetes 네트워킹을 연결해서 이해하기

## 핵심 질문

- Pod에 IP는 어떻게 할당되는가?
- Pod to Pod 통신 흐름은 정확히 어떻게 진행되는가?
- CNI는 정확히 어떤 일을 하고 있는가? 왜 필요한가?
- CNI를 쓰지 않으면 어떤 문제가 발생하는가?
- 각 CNI는 어떤 구조와 장단점을 가지고 있는가?
- Multi CNI와 멀티 네트워크는 어떤 상황에서 필요한가?

## 스터디 구성

- 각 주차마다 하나의 디렉터리를 사용합니다.
- 주차별 README에서 학습 내용, 정리, 질문, 참고 자료를 기록합니다.
- 브랜치 분리 없이 `main` 브랜치에서 작업하는 것을 기본으로 합니다.
- .md 이름은 **{name}-week1.md** 형식으로 합니다.
- 단, 개인별 실험/발표 자료를 따로 관리하고 싶다면 개별 브랜치 생성도 가능합니다.

## 커리큘럼

> 세부 내용에 있는 모든 걸 학습하기보다, 주제에 대해 학습할 때 참고하실 내용들이라고 생각해 주시면 됩니다. 예를 들어 week 1에서 리눅스 쪽 말고 Kubernetes 컴포넌트 Overview에 대해서만 더 deep dive 하고 싶다 하셔도 좋습니다. 주제에 맞는 내용들이라면 모두 좋습니다.

| Week | 주제 | 세부 내용 |
| --- | --- | --- |
| Week 1 | Kubernetes & Linux Networking Overview | - Kubernetes 컴포넌트 Overview<br>- Linux Network Namespace / veth / bridge<br>- TCP/IP Stack<br>- Routing / ARP<br>- Netfilter / iptables 등 |
| Week 2 | Kubernetes Networking & CNI Fundamentals | - Kubernetes Networking Model<br>- kubelet / CRI / CNI 동작 흐름<br>- Pod Network / Pod CIDR / IPAM<br>- kube-proxy와 Service<br>- Service → Pod Packet Flow 추적 |
| Week 3 | CNI Overview & Cloud Native Networking Landscape | - CNI의 역할과 구조<br>- CNI Plugin / IPAM<br>- Primary CNI Overview<br>- Flannel / Calico / Cilium Overview |
| Week 4 | Primary CNI | - Flannel Architecture / VXLAN<br>- Calico Architecture / Routing / BGP<br>- Overlay vs Underlay<br>- Pod-to-Pod Packet Flow 비교<br>- NetworkPolicy / IPAM |
| Week 5 | Cilium & eBPF | - Cilium Architecture<br>- Cilium Datapath<br>- eBPF 기본 개념 및 Hook<br>- kube-proxy Replacement<br>- NetworkPolicy / Identity<br>- Hubble Overview |
| Week 6 | Multi-Networking & Secondary Networking | - Multus Architecture / CNI Chaining<br>- Primary / Secondary Network<br>- NetworkAttachmentDefinition<br>- macvlan / ipvlan / SR-IOV<br>- Multi-NIC Pod 구성 및 Packet Flow |
| Week 7 | Cloud Native Networking 비교 & Use Cases | - Flannel / Calico / Cilium 비교<br>- Multus 기반 Networking 구성 비교<br>- macvlan / ipvlan / SR-IOV 비교<br>- Routing / Overlay / eBPF / Hardware Networking 비교<br>- Kubernetes / CNF 등 Use Case 분석 |
| Week 8 | Kubernetes Networking Common Issue Troubleshooting & Operations | - Pod-to-Pod / Pod-to-Service / External 통신 장애 분석<br>- CNI 장애 Troubleshooting<br>- IPAM / IP Pool 문제 분석<br>- Routing / NetworkPolicy / DNS / kube-proxy 문제 분석 |

## 기본 규칙

- 매주 학습 정리에는 다음 항목을 포함합니다.
  - 핵심 개념 정리
  - 질문 2개 이상
  - 참고 자료/링크
  - 필요한 경우 실습/패킷 흐름 그림
- 발표는 5분 정도씩 돌아가며 진행합니다.
- 각 사람은 자신이 학습한 내용을 README에 남겨 공동 학습 자료로 활용합니다.

## 참고 자료

- Kubernetes 공식 문서
- Linux 네트워크 관련 문서
- Cilium, Calico, Flannel 공식 문서
- Multus, macvlan, ipvlan, SR-IOV 관련 문서
- 관련 블로그/기술 아티클

## 운영 방식 요약

- 스터디 진행: 온라인, 매주 정기 모임
- 발표형태: 학습 내용 공유 + 토론
- 기록형태: 주차별 README 작성
- 협업형태: `main`브랜치 기반으로 문서 정리
