# Week 2. Kubernetes Networking & CNI Fundamentals

## 1. 학습 목표

Pod가 생성될 때 네트워크가 구성되는 과정과 Service 트래픽이 실제 Pod까지 전달되는 흐름을 이해한다.

- Kubernetes Network Model
- kubelet, CRI, CNI의 동작 흐름
- Pod Network, Pod CIDR, IPAM
- Service와 EndpointSlice
- kube-proxy와 Service → Pod 패킷 흐름

> Kubernetes는 네트워크의 요구사항을 정의하고, 실제 Pod 네트워크는 CNI 플러그인이 구현한다. Service 전달 방식 역시 kube-proxy 모드나 CNI 구현에 따라 달라질 수 있다.

---

## 2. Kubernetes Network Model

Kubernetes의 네트워크 모델은 다음을 기본으로 한다.

- 각 Pod는 클러스터 안에서 고유한 IP를 가진다.
- 같은 Pod의 컨테이너는 네트워크 환경을 공유하고 `localhost`로 통신한다.
- 서로 다른 Pod는 같은 Node인지와 관계없이 Pod IP로 통신할 수 있어야 한다.
- Pod 간 통신에는 원칙적으로 NAT가 필요하지 않아야 한다.
- Node의 agent는 해당 Node의 Pod와 통신할 수 있어야 한다.

-> [https://kubernetes.io/docs/concepts/services-networking/](https://kubernetes.io/docs/concepts/services-networking/)

Kubernetes가 직접 모든 네트워크 기능을 구현하는 것은 아니다. Linux 환경에서는 일반적으로 Container Runtime이 CNI 플러그인을 이용해 이 모델을 구현한다.

통신 종류에 따라 담당 영역을 나누면 다음과 같다.


| 통신                            | 주로 담당하는 영역                            |
| ----------------------------- | ------------------------------------- |
| 같은 Pod의 Container ↔ Container | 공유 Network Namespace                  |
| Pod ↔ Pod                     | CNI가 구성한 Pod Network                  |
| Pod ↔ Service                 | kube-proxy 또는 이를 대신하는 네트워크 구현         |
| 외부 ↔ Service                  | NodePort, LoadBalancer, Gateway 등의 기능 |


---

## 3. Pod가 생성될 때 네트워크가 만들어지는 흐름

Pod 생성과 네트워크 구성을 단순화하면 다음 순서로 진행된다.

```text
Pod 생성 요청
  → Scheduler가 Node 선택
  → 해당 Node의 kubelet이 Pod 감지
  → kubelet이 CRI로 Container Runtime에 요청
  → Runtime이 Pod Sandbox와 Network Namespace 준비
  → Runtime이 CNI Plugin 실행
  → CNI가 Interface, IP, Route 구성
  → Pod의 Container 실행
  → Pod IP가 Kubernetes API에 보고됨
```

### 1. Pod 선언과 배치

사용자가 Pod를 생성하면 API Server가 선언된 상태를 저장한다. Scheduler는 아직 Node가 정해지지 않은 Pod를 찾아 실행할 Node를 선택한다.

Scheduler가 직접 Pod를 실행하는 것은 아니다. 선택 결과를 API에 기록하면 해당 Node의 kubelet이 이를 확인한다.

### 2. kubelet과 CRI

kubelet은 컨테이너를 직접 실행하지 않고 **CRI(Container Runtime Interface)**를 통해 Container Runtime에 Pod 실행을 요청한다.

Runtime은 Pod의 컨테이너들이 공유할 sandbox와 Network Namespace를 준비한다. 이 공간이 준비되어야 CNI가 그 안에 network interface를 만들 수 있다.

### 3. CNI 실행

일반적인 Linux 환경에서 Runtime은 설정에 맞는 CNI 플러그인을 실행한다. CNI는 다음 작업을 수행한다.

- Pod Network Namespace에 interface 생성
- IPAM을 통한 Pod IP 확보
- Interface에 IP 할당
- 필요한 route 설정
- Host 측 network path 구성

CNI 작업이 끝나면 Runtime과 kubelet을 거쳐 Pod IP가 Kubernetes 상태에 반영된다.

---

## 4. CNI의 역할

CNI(Container Network Interface)는 특정 네트워크 제품이 아니라 **Runtime이 Network Plugin을 실행하는 방법을 정의한 명세**다.

Runtime은 대상 Network Namespace, Interface 이름, Network 설정 등을 CNI Plugin에 전달한다. Plugin은 요청에 따라 Pod를 Network에 연결하거나 연결에 사용한 자원을 정리한다.

`ADD`, `DEL`, `CHECK`는 `kubectl` 명령이나 Kubernetes API가 아니다. Container Runtime이 CNI Plugin을 실행할 때 `CNI_COMMAND`라는 값으로 전달하는 **작업 종류**다.

```text
Pod Network 생성
  → Runtime이 CNI Plugin에 ADD 요청

Pod 실행 중
  → 필요하면 CHECK로 Network 상태 확인

Pod Network 제거
  → Runtime이 CNI Plugin에 DEL 요청
```

- `ADD`: Pod가 사용할 Network Namespace에 interface를 만들거나 기존 interface를 설정한다. IPAM에서 IP를 받아 할당하고 route를 추가하는 작업도 이 과정에서 수행할 수 있다.
- `DEL`: `ADD`에서 만든 연결을 해제한다. Interface 제거, IP 반환, route와 관련 규칙 정리 등이 포함될 수 있다.
- `CHECK`: `ADD` 결과로 존재해야 하는 interface, IP, route 등이 올바른 상태인지 검사한다. 잘못된 구성을 직접 복구하는 명령은 아니다.

여기서 CNI 명세가 말하는 `Container`는 단순히 애플리케이션 컨테이너 하나만을 뜻하지 않는다. 네트워크에 연결할 **격리 영역**을 의미하며, Kubernetes에서는 일반적으로 Pod Sandbox의 Network Namespace에 해당한다.

CNI가 담당하는 범위를 구분하는 것도 중요하다.


| CNI가 담당하는 것              | 일반적으로 CNI의 직접 책임이 아닌 것 |
| ------------------------ | ---------------------- |
| Pod Interface와 IP 구성     | Pod Scheduling         |
| Pod Network 연결           | Service Backend 선택     |
| Pod Route 구성             | Cluster DNS Record 생성  |
| 필요에 따른 Network Policy 구현 | 애플리케이션의 L7 Routing     |


---

## 5. Pod Network와 IP 대역

Kubernetes 클러스터에서는 주로 세 종류의 IP를 볼 수 있다.


| 구분                        | 용도                      |
| ------------------------- | ----------------------- |
| Node IP                   | 실제 Node를 식별하는 주소        |
| Pod IP / Pod CIDR         | Pod에 할당하는 주소와 그 주소 대역   |
| Service IP / Service CIDR | Service의 가상 IP와 그 주소 대역 |


이 주소 대역들은 서로 겹치지 않도록 구성해야 한다.

### Pod CIDR

Pod CIDR은 Pod IP를 할당하기 위해 사용하는 주소 범위다. 구현에 따라 전체 Pod 대역을 Node별 작은 대역으로 나눌 수 있다.

```text
Cluster Pod CIDR: 10.244.0.0/16

Node A: 10.244.1.0/24
Node B: 10.244.2.0/24
```

이 경우 Node A의 Pod는 `10.244.1.0/24`, Node B의 Pod는 `10.244.2.0/24`에서 IP를 받을 수 있다.

하지만 모든 CNI가 Kubernetes의 Node별 Pod CIDR을 사용하는 것은 아니다. 일부 CNI는 자체 IP Pool과 IPAM을 사용한다. 따라서 실제 Pod IP의 출처는 사용하는 CNI 설정을 함께 확인해야 한다.

### Service CIDR

Service CIDR은 ClusterIP를 할당하는 대역이다. ClusterIP는 일반적인 Pod IP처럼 특정 애플리케이션 interface에 직접 붙어 있는 주소가 아니라, Service 전달 규칙에서 사용하는 **Virtual IP**다. (가상 IP)

---

## 6. IPAM

IPAM(IP Address Management)은 Pod에 사용할 IP를 선택하고, 사용 중인 주소가 중복 할당되지 않도록 관리한다.

CNI에서는 Network Interface를 구성하는 작업과 IP를 관리하는 작업을 분리할 수 있다. Main CNI Plugin이 별도의 IPAM Plugin을 호출하고, 전달받은 IP와 route를 Pod interface에 적용하는 방식이다.

```text
Container Runtime
  → Main CNI Plugin
      → IPAM Plugin
          → 사용할 IP와 Route 반환
      → Pod Interface에 적용
```

공식 CNI Plugin 저장소의 대표 IPAM 방식은 다음과 같다.


| IPAM         | 방식                             |
| ------------ | ------------------------------ |
| `host-local` | Node의 local database에 할당 상태 저장 |
| `static`     | 설정으로 지정한 고정 IP 사용              |
| `dhcp`       | DHCP Server로부터 IP 임대           |


실제 CNI 제품은 자체 IPAM을 사용할 수도 있다. 중요한 점은 **CNI가 IP를 interface에 설정하는 것**과 **IPAM이 사용할 IP를 선택하고 관리하는 것**을 구분하는 것이다.

---

## 7. Service와 EndpointSlice

Pod는 생성과 삭제에 따라 IP가 바뀔 수 있다. 클라이언트가 매번 새로운 Pod IP를 알아내기는 어렵기 때문에 Kubernetes는 **Service**라는 안정적인 접근 지점을 제공한다.

다음 Service를 예로 들어 보자.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
```

- `selector`: Service가 대상으로 삼을 Pod 선택
- `port`: 클라이언트가 Service에 접속할 Port
- `targetPort`: 실제 Backend Pod의 Port
- `clusterIP`: 클러스터 내부에서 사용하는 Service Virtual IP

### EndpointSlice

EndpointSlice는 Service 뒤에 연결된 실제 Network Endpoint 정보를 저장한다.

```text
Service
  selector: app=web
  ClusterIP: 10.96.0.10:80
             │
             │ Selector와 일치하는 Pod 확인
             ↓
EndpointSlice
  ├─ Pod A: 10.244.1.2:8080
  └─ Pod B: 10.244.2.3:8080
```

Service의 selector와 일치하는 Pod가 생성되거나 삭제되면 EndpointSlice의 Backend 목록도 변경된다. Service는 **어떤 Pod를 선택할지** 정의하고, EndpointSlice는 **현재 전달할 수 있는 실제 Pod IP와 Port**를 기록한다.

따라서 클라이언트는 변하는 Pod IP를 직접 찾지 않고 일정한 Service 주소를 사용할 수 있다.

---

## 8. kube-proxy의 역할

kube-proxy는 Service와 EndpointSlice의 변화를 확인하고, Service 트래픽을 Backend Endpoint로 전달하기 위한 Node의 데이터 경로를 구성한다.

이름과 달리 모든 packet을 kube-proxy process가 직접 받아 전달하는 것은 아니다. 일반적으로 kube-proxy는 Linux Kernel이 packet을 처리할 수 있도록 규칙을 설정한다.

대표적인 구현 방식에는 iptables, IPVS, nftables 등이 있다. 일부 네트워크 구현은 kube-proxy 없이 Service 기능을 직접 제공하기도 한다.


| 구성 요소         | 역할                                      |
| ------------- | --------------------------------------- |
| CNI           | Pod IP와 Pod-to-Pod Network 구성           |
| EndpointSlice | Service의 Backend IP와 Port 정보 제공         |
| kube-proxy    | Service IP를 Backend Endpoint로 전달할 규칙 구성 |


즉, kube-proxy가 Service 주소를 실제 Pod 주소로 바꾸더라도 CNI가 구성한 Pod Network가 없다면 그 Pod까지 packet을 전달할 수 없다.

---

## 9. Service → Pod Packet Flow

Service가 Backend Pod로 연결되는 과정은 **전달 규칙을 준비하는 단계**와 **실제 Packet이 이동하는 단계**로 나눠 보면 이해하기 쉽다.

아래는 iptables 기반 kube-proxy를 단순화한 예시다.

```text
Client Pod:       10.244.1.5
Service:          10.96.0.10:80
Selected Backend: 10.244.2.3:8080
```

### 1. Service 규칙 준비

```text
Service 생성
  selector: app=web
        ↓
Selector와 일치하는 Pod 확인
        ↓
EndpointSlice 갱신
  ├─ 10.244.1.2:8080
  └─ 10.244.2.3:8080
        ↓
kube-proxy가 Service와 EndpointSlice 확인
        ↓
각 Node에 Service 전달 규칙 구성
```

이 단계에서는 아직 사용자 Packet이 흐르지 않는다. kube-proxy는 Service와 EndpointSlice의 변경을 확인하고 Kernel이 사용할 규칙을 미리 구성한다.

### 2. 요청 Packet 전달

Client Pod는 Backend Pod의 주소를 알 필요가 없다. 항상 Service 주소로 요청한다.

```text
① Client Pod
   src: 10.244.1.5
   dst: 10.96.0.10:80
          │
          │ veth와 CNI Network
          ↓
② Client Node의 Kernel
   Service 규칙과 일치
          │
          │ Backend 하나 선택 후 DNAT
          │ dst: 10.96.0.10:80
          │        → 10.244.2.3:8080
          ↓
③ Routing
   변경된 Backend Pod IP를 기준으로 경로 결정
          │
          │ CNI가 구성한 Pod Network
          ↓
④ Backend Pod
   10.244.2.3:8080에서 요청 수신
```

핵심은 Node의 Service 규칙이 Packet의 destination을 다음과 같이 변경한다는 점이다.

**DNAT(Destination Network Address Translation)**는 Packet의 목적지 IP나 Port를 다른 값으로 변환하는 방식이다. 여기서는 Client가 요청한 Service 주소 `10.96.0.10:80`을 실제 Backend Pod 주소 `10.244.2.3:8080`으로 변경한다.


| 처리 시점         | Source       | Destination       |
| ------------- | ------------ | ----------------- |
| Client가 요청할 때 | `10.244.1.5` | `10.96.0.10:80`   |
| DNAT 이후       | `10.244.1.5` | `10.244.2.3:8080` |


Destination이 Backend Pod IP로 바뀐 다음부터는 일반적인 Pod-to-Pod 통신이다. Kernel은 Routing을 수행하고, CNI가 구성한 경로를 이용해 같은 Node 또는 다른 Node의 Backend Pod로 전달한다.

### 3. 응답 Packet 전달

```text
Backend Pod
  → Client Pod 방향으로 응답
  → Node가 기존 Connection의 변환 정보 확인
  → DNAT의 반대 방향 변환 적용
  → Client Pod는 Service IP에서 응답받은 것으로 인식
```

요청 Packet에 적용한 주소 변환 정보는 connection tracking에 기록된다. 이를 이용해 응답 방향에서는 source가 Backend Pod IP에서 Service IP로 되돌아온 것처럼 처리된다.

> 위 흐름은 일반적인 Cluster 내부 통신을 단순화한 것이다. Traffic Policy, Hairpin 통신, 외부 접근 방식에 따라 SNAT이 추가될 수 있다.
>
> **SNAT(Source Network Address Translation)**는 반대로 Packet의 출발지 IP나 Port를 다른 값으로 변환하는 방식이다. Kubernetes의 기본 Pod-to-Pod 통신에서는 필요하지 않지만, 외부 통신이나 일부 Service 경로에서는 응답이 돌아올 경로를 보장하기 위해 사용될 수 있다.

Service 객체나 kube-proxy process가 Packet을 직접 중계하는 것은 아니다. **Service는 안정적인 주소를 제공하고, EndpointSlice는 Backend를 기록하며, kube-proxy는 규칙을 만들고, 실제 전달은 Kernel과 CNI Network가 수행한다.**

실제 Packet 처리 위치와 Backend 선택 방식은 kube-proxy의 iptables, IPVS, nftables 모드 또는 eBPF 기반 구현에 따라 달라질 수 있다.

---

## 10. 전체 흐름 연결

Pod 생성부터 Service 연결까지 이어 보면 다음과 같다.

```text
1. Pod가 Node에 배치됨
       ↓
2. Runtime이 Pod Sandbox와 Network Namespace 준비
       ↓
3. CNI와 IPAM이 Interface, Pod IP, Route 구성
       ↓
4. Pod IP가 Kubernetes API에 보고됨
       ↓
5. Service의 EndpointSlice에 Pod가 Backend로 반영됨
       ↓
6. kube-proxy가 Service 전달 규칙 갱신
       ↓
7. Service Traffic이 선택된 Pod로 전달됨
```

각 요소의 핵심 책임은 다음과 같다.


| 요소            | 핵심 질문                                |
| ------------- | ------------------------------------ |
| CRI           | kubelet이 Runtime에 Pod 실행을 어떻게 요청하는가? |
| CNI           | Pod를 Network에 어떻게 연결하는가?             |
| IPAM          | Pod가 사용할 IP를 어떻게 선택하고 관리하는가?         |
| EndpointSlice | Service가 현재 어떤 Backend를 가지는가?        |
| kube-proxy    | Service Traffic을 Backend로 어떻게 전달하는가? |


---

## 11. 추가 조사: 멀티 노드 환경의 Pod 통신

지금까지는 주로 같은 Node 안에서 `Network Namespace → veth → Bridge 또는 Routing → veth`로 이어지는 통신을 살펴봤다. 그렇다면 목적지 Pod가 다른 Node에 있으면 Packet은 어떻게 이동할까?

Kubernetes Network Model은 서로 다른 Node의 Pod도 NAT 없이 Pod IP로 통신할 수 있어야 한다고 요구한다. 하지만 Kubernetes가 구체적인 전달 방법까지 정하지는 않으며, 실제 경로는 CNI가 구성한다.

다음과 같은 클러스터를 가정한다.

```text
Node A
  Node IP: 192.168.10.11
  Pod CIDR: 10.244.1.0/24
  Pod A:    10.244.1.2

Node B
  Node IP: 192.168.10.12
  Pod CIDR: 10.244.2.0/24
  Pod B:    10.244.2.3
```

Pod A가 Pod B로 Packet을 보내면 다음 과정이 필요하다.

```text
Pod A: 10.244.1.2
  → veth
  → Node A의 Routing Table
  → 10.244.2.0/24가 Node B에 있다는 것을 확인
  → Node 사이의 Network를 통해 전달
  → Node B에서 Pod B의 veth 방향으로 Routing
  → Pod B: 10.244.2.3
```

핵심은 각 Node가 **목적지 Pod CIDR이 어느 Node에 있는지** 알 수 있어야 한다는 점이다. CNI는 이 문제를 크게 Routing 방식이나 Overlay 방식으로 해결할 수 있다.

### Routing 방식

Node의 Routing Table 또는 기반 Network가 다른 Node의 Pod CIDR로 가는 경로를 알고 있는 방식이다.

```text
Node A Routing Table
10.244.2.0/24 → Node B 방향

Pod A Packet
src: 10.244.1.2
dst: 10.244.2.3
        ↓
추가 Tunnel Header 없이 Node B로 전달
```

- Packet의 Pod IP를 그대로 Routing한다.
- 추가 캡슐화가 없어 Header overhead가 적다.
- Node 또는 Underlay Network가 Pod CIDR의 경로를 알아야 한다.

### Overlay 방식

물리 Network가 Pod CIDR을 몰라도 되도록 원래 Pod Packet을 Node 사이에서 전달 가능한 Packet 안에 넣는 방식이다. 대표적인 예가 VXLAN이다.

```text
Outer Packet
src: Node A IP 192.168.10.11
dst: Node B IP 192.168.10.12

┌─────────────────────────────────┐
│ Inner Packet                    │
│ src: Pod A IP 10.244.1.2        │
│ dst: Pod B IP 10.244.2.3        │
└─────────────────────────────────┘
```

1. Node A가 원래 Pod Packet을 VXLAN Packet으로 캡슐화한다.
2. Underlay Network는 바깥쪽 Node IP를 보고 Node B까지 전달한다.
3. Node B가 바깥쪽 Header를 제거한다.
4. 안쪽 Pod Packet을 Pod B로 Routing한다.

안쪽 Packet의 Pod IP는 유지되므로 Pod A와 Pod B는 중간의 Node IP나 Tunnel을 알 필요가 없다.

### 같은 Node와 다른 Node 비교


| 구분                        | 같은 Node                      | 다른 Node                      |
| ------------------------- | ---------------------------- | ---------------------------- |
| Node의 물리 Network 사용       | 보통 사용하지 않음                   | 사용함                          |
| 주요 경로                     | veth, Bridge 또는 Host Routing | Host Routing과 Node 간 Network |
| Tunnel Header             | 필요 없음                        | Overlay 방식이면 추가됨             |
| 목적지 Node 탐색               | 필요 없음                        | Pod CIDR과 Node의 관계가 필요함      |
| Pod Source/Destination IP | 유지                           | 기본 Network Model에서는 유지       |


정리하면 Week 1의 veth와 Routing이 사라지는 것이 아니다. 멀티 노드에서는 그 경로 중간에 **목적지 Node를 찾는 과정과 Node 사이의 전달 구간**이 추가된다.

```text
같은 Node
Pod → veth → Bridge/Host Routing → veth → Pod

다른 Node
Pod → veth → Node A Routing
    → Direct Routing 또는 Overlay Tunnel
    → Node B Routing → veth → Pod
```

---

## 12. 학습하면서 생긴 질문

### Q1. Service의 Selector와 일치하는 Pod라도 아직 Ready 상태가 아니라면 EndpointSlice와 실제 트래픽 전달에는 어떻게 반영될까?

### Q2. Cilium처럼 kube-proxy를 사용하지 않는 구성에서는 어떤 컴포넌트가 Service와 EndpointSlice를 확인하고 전달 경로를 만들까?

---

## 13. 용어 설명집


| 용어                 | 설명                                                            |
| ------------------ | ------------------------------------------------------------- |
| CNI                | Runtime과 Network Plugin 사이의 실행 방법과 결과 형식을 정의한 명세              |
| CNI Plugin         | CNI 명세에 따라 interface, IP, route 등 실제 Network 구성을 적용하는 프로그램    |
| `CNI_COMMAND`      | Runtime이 Plugin에 요청할 작업 종류를 전달하는 값                            |
| `ADD`              | Pod의 격리된 Network 영역을 특정 Network에 연결하는 작업                      |
| `DEL`              | 기존 Network 연결과 관련 자원을 정리하는 작업                                 |
| `CHECK`            | 기존 Network 구성이 예상한 상태인지 검사하는 작업                               |
| CRI                | kubelet과 Container Runtime 사이의 표준 Interface                   |
| Container Runtime  | Pod Sandbox와 Container의 실행 및 생명주기를 관리하는 소프트웨어                 |
| Pod Sandbox        | 한 Pod의 Container들이 공유하는 실행·Network 환경의 기반                     |
| Network Namespace  | Interface, IP, Route 등 Linux Network 자원을 격리하는 기능              |
| Pod Network        | Pod IP를 이용해 Pod들이 서로 통신할 수 있도록 구성된 Network                    |
| Pod CIDR           | Pod IP를 할당하기 위해 사용하는 주소 범위                                    |
| Underlay Network   | Node 사이를 실제로 연결하는 물리 또는 기반 Network                            |
| Overlay Network    | 기존 Network 위에 Tunnel을 만들어 구성한 논리 Network                      |
| Encapsulation      | 원래 Packet을 다른 Packet의 Payload 안에 넣는 처리                        |
| VXLAN              | Ethernet Frame을 UDP Packet 안에 넣어 전달하는 Overlay Tunnel 방식       |
| IPAM               | 사용할 IP를 선택하고 중복되지 않도록 할당·반환 상태를 관리하는 기능                       |
| Service            | 변할 수 있는 여러 Backend Pod에 안정적인 접근 지점을 제공하는 Kubernetes 객체        |
| ClusterIP          | Cluster 내부에서 Service에 접근하기 위해 사용하는 Virtual IP                 |
| Service CIDR       | ClusterIP를 할당하기 위해 사용하는 주소 범위                                 |
| EndpointSlice      | Service에 연결된 실제 Backend IP와 Port 정보를 담는 객체                    |
| kube-proxy         | Service와 EndpointSlice를 확인하고 Service Traffic 전달 규칙을 구성하는 컴포넌트 |
| DNAT               | Packet의 Destination IP 또는 Port를 다른 값으로 변환하는 방식                |
| SNAT               | Packet의 Source IP 또는 Port를 다른 값으로 변환하는 방식                     |
| Backend / Endpoint | Service가 Traffic을 전달할 실제 목적지 Pod IP와 Port                     |


---

## 14. 참고 자료

- [Kubernetes Network Model](https://kubernetes.io/docs/concepts/services-networking/)
- [Kubernetes Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Kubernetes Service](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)
- [CNI Specification](https://github.com/containernetworking/cni/blob/main/SPEC.md)
- [CNI Plugins](https://github.com/containernetworking/plugins)
- [VXLAN — Linux Kernel Documentation](https://docs.kernel.org/networking/vxlan.html)
