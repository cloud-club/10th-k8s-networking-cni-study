# Kubernetes 컴포넌트 Overview
https://kubernetes.io/ko/docs/concepts/overview/components/
https://kubernetes.io/ko/docs/concepts/architecture/
![kubernetes-cluster-architecture](./images/kubernetes-cluster-architecture-SY.svg)
- Control Plane: `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`, `cloud-controller-manager`
- Node: `kubelet`, `kube-proxy`, Container Runtime 
## 핵심 컨트롤러
쿠버네티스 클러스터는 컨트롤 플레인과 하나 이상의 워커 노드로 구성된다.
### 컨트롤 플레인 컴포넌트
- kube-apiserver: 쿠버네티스 HTTP API를 노출하는 핵심 서버 컴포넌트
- etcd: 모든 API 서버 데이터를 위한 일관성과 고가용성을 갖춘 키-값 저장소
- kube-scheduler: 아직 노드에 할당되지 않은 파드를 찾아 적절한 노드에 할당
- kube-controller-manager: 컨트롤러를 실행하여 쿠버네티스 API 동작을 구현
- cloud-controller-manager(선택): 기본 클라우드 공급자와 통합

## 노드 컴포넌트
모든 노드에서 실행되며, 실행 중인 파드를 유지하고 쿠버네티스 런타임 환경을 제공한다.
- kubelet: 파드와 그 안의 컨테이너가 실행 중임을 보장
- kube-proxy(선택): 노드에서 네트워크 규칙을 유지하여 서비스를 구현
- 컨테이너 런타임: 컨테이너 실행을 담당하는 소프트웨어

## 애드온
애드온은 쿠버네티스 기능을 확장한다. 아래는 예시.
- DNS: 클러스터 전반의 DNS 해석을 담당
- 웹 UI(대시보드): 웹 인터페이스를 통한 클러스터 관리를 제공
- 컨테이너 리소스 모니터링: 컨테이너 매트릭을 수집하고 저장
- 클러스터-레벨 로깅: 컨테이너 로그를 중앙 로그 저장소에 저장


# Linux Network Namespace / veth / bridge
## Network Namespace
호스트 입장에서는 모든 프로세스가 보이지만, namespace 안에서는 자기 자신만 보인다. 네트워크 namespace는 이 격리를 네트워크 레벨에 적용한 것 — 자기만의 가상 인터페이스, 라우팅 테이블, ARP 테이블을 가지며 호스트의 실제 네트워크 정보는 보이지 않는다.

## veth pair (가상 케이블)
두 namespace를 veth pair(virtual ethernet pair)로 직접 연결 가능.
연결하면 두 namespace가 서로 ping이 가능해지는데, 이 통신은 호스트의 ARP 테이블에 전혀 나타나지 않는다 — namespace 내부 통신이라 호스트는 알 수 없다.

## Linux Bridge (가상 스위치)
namespace가 2개 이상이면 직접 케이블로 다 잇기 번거롭다. → 가상 스위치(Linux Bridge)를 만들어서 모두 연결.
이 구조가 반복되면 여러 namespace가 모두 같은 브리지에 연결되어 서로 통신이 가능하다.

## Kubernetes 개념과 연결지으면?
- Network Namespace - Pod마다 격리된 네트워크 환경의 기반
- veth pair, bridge - CNI 플러그인이 Pod를 클러스터 네트워크에 연결하는 방식


# TCP/IP Stack
OSI 계층과의 대응

| 계층             | 담당          | Pod 네트워킹에서의 역할                                                       |
| -------------- | ----------- | -------------------------------------------------------------------- |
| L2 - Data Link | 이더넷, MAC 주소 | veth pair/bridge가 프레임을 주고 받는 계층. 같은 브리지에 붙은 인터페이스끼리는 MAC 주소로 통신      |
| L3 - Network   | IP, 라우팅     | Pod IP 할당, 서로 다른 노드/서브넷 간 라우팅 결정이 이뤄지는 계층                            |
| L4 - Transport | TCP, UDP    | 포트 번호로 프로세스를 구분. Service의 ClusterIP: Port, kube-proxy의 DNAT가 개입하는 지점 |
이 3계층은 서로 독립적으로 동작하는게 아니라, 하나의 패킷이 이 계층들을 순서대로 통과하면서 각 계층에 맞는 헤더를 씌우고 벗기는 과정을 거친다. 이게 바로 캡슐화와 역캡슐화이다.
### 캡슐화(Encapsulation) / 역캡슐화(Decapsulation)
보내는 쪽
```
[Application Data]
   ↓ L4: TCP 헤더 추가 (출발/도착 포트, 시퀀스 번호 등)
[TCP Header | Data]
   ↓ L3: IP 헤더 추가 (출발/도착 IP)
[IP Header | TCP Header | Data]
   ↓ L2: 이더넷 헤더 추가 (출발/도착 MAC)
[Ethernet Header | IP Header | TCP Header | Data]
```

받는 쪽
```
[Ethernet Header | IP Header | TCP Header | Data]
   ↓ L2에서 이더넷 헤더 확인 → 내 MAC이면 통과
   ↓ L3에서 IP 헤더 확인 → 내 IP면 통과, 아니면 라우팅
   ↓ L4에서 TCP 헤더 확인 → 포트로 프로세스에 전달
[Application Data] → 최종 목적지 프로세스
```

### Pod 간 통신 시 계층별로 실제 일어나는 일
**같은 노드 내 Pod A → Pod B (앞서 다룬 netns/veth/bridge 흐름과 연결)**

1. Pod A의 애플리케이션이 소켓에 데이터를 씀 → 커널이 TCP/IP 헤더를 씌움
2. 패킷이 Pod A의 netns 안 veth(예: `veth-red`)로 나감
3. veth pair 반대쪽(`veth-red-br`)이 브리지(`cni0` 등)에서 나타남 → **L2에서 목적지 MAC 확인**
4. 브리지가 Pod B 쪽 veth로 프레임을 전달 (같은 L2 세그먼트라 라우팅 불필요, 스위칭만 발생)
5. Pod B의 netns 안에서 역캡슐화 → 애플리케이션이 데이터 수신

**다른 노드 간 Pod A → Pod C (여기서 L3 라우팅이 개입)**

1. Pod A → 브리지까지는 위와 동일
2. 브리지는 목적지 IP(Pod C의 IP)가 자기 서브넷 밖이라는 걸 인지 → **L3 라우팅 테이블 조회**
3. 노드의 라우팅 테이블에 따라 물리 NIC로 패킷 전달 (또는 CNI가 오버레이라면 VXLAN 캡슐화 후 UDP로 전송)
4. 상대 노드가 패킷을 받아 역캡슐화 → 자기 브리지로 전달 → Pod C의 veth로 전달


# Routing / ARP
## Routing Table
- 커널이 패킷을 보낼 때마다 조회하는 "목적지별 경로" 테이블
- `ip route` 로 확인
	- 특정 대역 라우트: 해당 대역은 지정된 인터페이스로 직접 전송. (같은 L2, 라우터 불필요)
	- default 라우트(기본 게이트웨이): 위에 안 걸리는 모든 트래픽을 보내는 곳.

### Pod 네트워킹 적용
- 같은 노드 Pod ↔ Pod: 로컬 브리지 대역 → L2 스위칭만으로 처리, 라우팅 테이블 조회 불필요
- 다른 노드 Pod ↔ Pod: 목적지가 로컬 대역 밖 → 라우팅 테이블 조회 → 물리 NIC 또는 터널 인터페이스로 전달
- CNI별 라우팅 테이블 구성 방식
    - Calico: BGP로 각 노드에 다른 노드의 Pod 서브넷 경로 전파
    - Flannel: 데몬이 라우팅 테이블/오버레이 인터페이스 직접 설정

## ARP (Address Resolution Protocol)
- IP(논리 주소) → MAC(물리 주소) 매핑을 알아내는 프로토콜
- 동작 순서
    1. 브로드캐스트: IP의 MAC 주소가 무엇인지
    2. 해당 IP 소유자가 유니캐스트로 자기 MAC 응답
    3. 요청자가 ARP 캐시에 매핑 저장


# Netfilter / iptables
## Netfilter
netfilter는 리눅스 커널 안에 내장된 패킷 필터링/변조 프레임워크이다. 
패킷이 커널 네트워크 스택을 지나갈 때 특정 지점마다 멈춰서 등록된 규칙을 확인하고 통가시킬지, 버릴지, 변조할지를 결정한다.

```
                    ┌─────────────┐
외부 → NIC 진입 → PREROUTING → 라우팅 결정
                    └─────────────┘
                          │
              ┌───────────┴───────────┐
	      목적지가 로컬인가?         목적지가 다른 곳인가?
              │                        │
            INPUT                   FORWARD
              │                        │
	        로컬 프로세스              (통과만)
              │                        │
            OUTPUT                     │
	     (로컬에서 생성된 패킷)             │
              │                        │
              └───────────┬────────────┘
                    POSTROUTING
                          │
                    NIC 송신 → 외부
```


| 훅           | 시점                        | Pod 네트워킹에서의 예                                  |
| ----------- | ------------------------- | ---------------------------------------------- |
| PREROUTING  | 패킷이 들어오자마자, 라우팅 결정 전      | Service ClusterIP → Pod IP DNAT가 여기서 일어남       |
| INPUT       | 라우팅 결정 후 목적지가 로컬 프로세스일 때  | 노드 자체를 향한 패킷                                   |
| FORWARD     | 라우팅 결정 후 목적지가 다른 곳(전달)일 때 | Pod 간 트래픽이 브리지를 거쳐 전달될 때                       |
| OUTPUT      | 로컬 프로세스가 만든 패킷이 나갈 때      | 노드 자체 프로세스가 보내는 트래픽                            |
| POSTROUTING | 패킷이 실제로 나가기 직전            | Pod → 외부 인터넷 egress 시 SNAT/Masquerade가 여기서 일어남 |

## iptables 테이블
iptables는 규칙을 테이블 별로 관리한다. 각 테이블은 위 5개의 훅 중 자기와 관련된 일부만 사용.

| 테이블          | 용도                    | 사용 가능한 훅                        |
| ------------ | --------------------- | ------------------------------- |
| filter (기본값) | 패킷 허용/차단 (방화벽)        | INPUT, FORWARD, OUTPUT          |
| nat          | 주소/포트 변환 (SNAT/DNAT)  | PREROUTING, OUTPUT, POSTROUTING |
| mangle       | 패킷 헤더 수정 (TTL, TOS 등) | 모든 훅                            |

### SNAT/DNAT
- DNAT(Destination NAT)
	- 목적지 IP/포트를 바꿈
	- 외부 요청이 ClusterIP/NodePort로 들어와서 실제 Pod IP로 바뀔 때 사용
- SNAT(Source NAT)
	- 출발지 IP/포트를 바꿈
	- Pod가 외부로 나갈 때 노드 IP로 위장(Masquerade)할 때 사용


---
## 질문

1. namespace가 딱 2개일 때도 브리지를 쓰는게 나은 경우가 있을까?
2. 같은 노드 내 Pod 통신은 라우팅 없이 L2 스위칭만으로 끝난다고 했는데, 그렇다면 브리지에 붙은 veth들끼리는 IP 주소가 왜 필요한가?