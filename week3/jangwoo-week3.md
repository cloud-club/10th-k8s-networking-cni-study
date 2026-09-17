# Week 3. CNI Overview & Cloud Native Networking Landscape

## 1. 학습 목표

Week 2에서 살펴본 Pod Network 생성 과정을 바탕으로 CNI의 구조와 대표적인 Kubernetes Network Solution을 비교한다.

- CNI가 표준 Interface로서 정의하는 범위
- CNI Plugin과 IPAM의 관계
- Primary CNI의 의미
- Flannel, Calico, Cilium의 기본 구조와 차이

> CNI는 표준이고, Flannel·Calico·Cilium은 그 표준을 이용해 Kubernetes Network를 구현하는 Solution이다.

---

## 2. Week 2 내용과 연결

Week 2에서는 Pod가 생성될 때 Network가 구성되는 전체 흐름을 살펴봤다.

```text
kubelet
  → CRI
  → Container Runtime
  → CNI Plugin 실행
  → Interface, IP, Route 구성
  → Pod Network 연결 완료
```

이번 주에는 이 중 `CNI Plugin` 부분을 조금 더 자세히 살펴본다.


| Week 2에서 다룬 내용           | Week 3에서 이어서 볼 내용                    |
| ------------------------ | ------------------------------------ |
| CNI가 Pod Network를 구성한다   | CNI가 정확히 무엇을 표준화하는가?                 |
| IPAM이 Pod IP를 관리한다       | Main Plugin과 IPAM Plugin은 어떻게 나뉘는가?  |
| 다른 Node의 Pod까지 경로가 필요하다  | 각 Network Solution은 그 경로를 어떻게 만드는가?  |
| CNI와 kube-proxy의 역할이 다르다 | 일부 Solution은 Service 처리까지 대신할 수 있는가? |


Kubernetes는 Network Model을 제시하지만, Pod Interface 생성이나 Node 간 Packet 전달 방식을 하나로 고정하지 않는다. 따라서 동일한 요구사항을 Flannel, Calico, Cilium이 서로 다른 구조로 구현할 수 있다.

---

## 3. CNI란?

CNI(Container Network Interface)는 **Container Runtime이 Network Plugin을 실행하는 방법을 정의한 명세**다.

Kubernetes 공식 문서에 따르면 Container Runtime은 Kubernetes Network Model을 구현하기 위해 필요한 CNI Plugin을 불러오도록 구성된다.

```text
Container Runtime
        │
        │ CNI가 정의한 입력과 실행 방식
        ▼
    CNI Plugin
        │
        └─ Interface, IP, Route 등 실제 Network 구성
```

### CNI가 정의하는 것

CNI Specification은 주로 다음 내용을 정의한다.

- Network 설정 형식
- Runtime이 Plugin을 호출하는 Protocol
- Plugin 실행과 위임 방식
- Plugin이 Runtime에 반환하는 결과 형식

반면 다음 항목의 구체적인 구현은 CNI가 하나로 정하지 않는다.

- Pod Packet을 다른 Node까지 전달하는 방식
- VXLAN, BGP 같은 Network 기술의 선택
- NetworkPolicy 구현 방식
- Service Load Balancing 구현 방식

따라서 다음 개념을 구분해야 한다.


| 구분                | 의미                                                           |
| ----------------- | ------------------------------------------------------------ |
| CNI Specification | Runtime과 Plugin 사이의 약속                                       |
| CNI Plugin        | CNI 명세에 따라 실제 Network 작업을 수행하는 실행 프로그램                       |
| Network Solution  | 여러 Plugin과 Node Agent 등을 조합해 Pod Network 전체를 구현하는 제품 또는 프로젝트 |


즉, **CNI는 Network 자체가 아니라 Network를 연결하기 위한 표준 Interface**다.

---

## 4. CNI의 기본 구조

CNI 환경은 단순화하면 다음 요소로 구성된다.


| 구성 요소                 | 역할                             |
| --------------------- | ------------------------------ |
| Container Runtime     | Network 설정을 읽고 CNI Plugin 실행   |
| Network Configuration | 사용할 Plugin과 옵션을 JSON 형식으로 정의   |
| CNI Plugin Binary     | Interface, IP, Route 등의 설정 수행  |
| Network Namespace     | Pod가 사용할 격리된 Network 공간        |
| Plugin Result         | 생성한 Interface, IP, Route 등의 결과 |


Pod가 생성될 때의 흐름은 다음과 같다.

```text
1. Runtime이 Pod Network Namespace 준비
2. Runtime이 CNI Network 설정 확인
3. Runtime이 CNI Plugin 실행
4. Plugin이 Interface와 Route 구성
5. 필요하면 IPAM Plugin을 호출해 IP 확보
6. Plugin이 구성 결과를 Runtime에 반환
```

### 대표적인 CNI Command

Week 2에서 살펴본 것처럼 CNI Command는 `kubectl` 명령이 아니라 Runtime이 Plugin을 실행할 때 전달하는 작업 종류다.


| Command | 의미                                 |
| ------- | ---------------------------------- |
| `ADD`   | Container를 Network에 연결하고 필요한 자원 구성 |
| `DEL`   | Network 연결과 할당된 자원 정리              |
| `CHECK` | 기존 Network 구성이 예상한 상태인지 검사         |


최신 CNI 명세에는 Plugin 준비 상태를 확인하는 `STATUS`, 오래된 자원을 정리하는 `GC`, 지원 버전을 확인하는 `VERSION`도 정의되어 있다. 다만 기본 생명주기를 이해할 때는 `ADD → CHECK → DEL`의 관계를 먼저 보면 된다.

### Plugin Chaining

CNI는 여러 Plugin을 순서대로 실행하는 구성을 지원한다. 한 Plugin이 모든 기능을 직접 구현할 필요는 없다.

```text
Container Runtime
  → Main Plugin: Pod Interface 연결
  → portmap: Host Port Mapping
  → bandwidth: Traffic 대역폭 제한
```

앞 Plugin의 결과는 다음 Plugin에 `prevResult`로 전달될 수 있다. 이를 통해 Interface를 만든 뒤 별도 Plugin이 Port Mapping이나 대역폭 제한을 추가할 수 있다.

CNI 프로젝트가 제공하는 Reference Plugin의 예시는 다음과 같다.


| 종류   | 예시                                 | 역할                   |
| ---- | ---------------------------------- | -------------------- |
| Main | `bridge`, `ptp`, `macvlan`         | Interface와 연결 방식 구성  |
| IPAM | `host-local`, `dhcp`, `static`     | IP 주소 할당             |
| Meta | `portmap`, `bandwidth`, `firewall` | 기존 Network 구성에 기능 추가 |


---

## 5. IPAM이 필요한 이유

IPAM(IP Address Management)은 사용할 IP를 선택하고, 할당 상태를 관리하며, 더 이상 사용하지 않는 IP를 회수하는 기능이다.

Interface만 생성한 상태에서는 Pod가 실제 Network로 통신할 수 없다. 최소한 다음 정보가 필요하다.

- Pod IP
- Subnet 또는 Prefix
- Gateway
- Route

```text
CNI Main Plugin
  │
  ├─ veth 등 Interface 생성
  │
  └─ IPAM Plugin 호출
       → 사용 가능한 IP 선택
       → IP와 Route 정보 반환
       → 할당 상태 기록
```

IPAM이 없다면 여러 Pod에 같은 IP가 할당되거나, 삭제된 Pod의 IP가 계속 사용 중인 것으로 남는 문제가 발생할 수 있다.


| IPAM 방식          | 특징                                     |
| ---------------- | -------------------------------------- |
| `host-local`     | 각 Node의 Local 저장소에서 IP 할당 상태 관리        |
| `dhcp`           | DHCP Server에서 IP 임대                    |
| `static`         | 설정에 지정된 고정 IP 사용                       |
| Solution 자체 IPAM | Calico, Cilium 등이 자체 주소 Pool과 할당 정책 관리 |


중요한 점은 **CNI와 IPAM이 같은 개념은 아니라는 것**이다. CNI Plugin이 IPAM Plugin에 IP 선정을 위임할 수도 있고, Network Solution이 자체 IPAM을 제공할 수도 있다.

---

## 6. Primary CNI란?

Primary CNI는 일반적으로 Cluster에서 **Pod의 기본 Network 연결을 담당하는 CNI 기반 Network Solution**을 뜻한다.

```text
일반 Pod 생성
  → Primary CNI
  → 기본 eth0, Pod IP, Route 구성
  → Cluster의 기본 Pod Network에 연결
```

예를 들어 Calico를 기본 Network로 설치했다면 대부분의 Pod는 생성될 때 Calico를 통해 Network에 연결된다. 이 경우 Calico를 Primary CNI라고 부를 수 있다.

다만 `PrimaryCNI`라는 Kubernetes API Object가 존재하는 것은 아니다. 이는 여러 Network가 존재할 수 있는 환경에서 **기본 Pod Network를 담당하는 Plugin**을 구분하기 위해 사용하는 생태계 용어에 가깝다.


| 구분            | 역할                                 |
| ------------- | ---------------------------------- |
| Primary CNI   | Pod의 기본 Interface와 기본 Network 구성   |
| Secondary CNI | 필요할 때 Pod에 추가 Network Interface 제공 |


이번 주에는 Primary CNI 역할을 수행할 수 있는 대표적인 Solution인 Flannel, Calico, Cilium을 비교한다.

---

## 7. 비교 전에 알아둘 개념

### Underlay와 Overlay

**Underlay Network**는 Node 사이를 실제로 연결하는 기반 Network다. 예를 들면 같은 VLAN으로 연결된 사내 Network나 Cloud VPC가 이에 해당한다.

**Overlay Network**는 Underlay 위에 Tunnel을 만들어 Pod용 논리 Network를 구성하는 방식이다. Underlay가 Pod IP의 경로를 모르더라도 원래 Packet을 Node IP Packet 안에 넣어 목적지 Node까지 보낼 수 있다. 이처럼 원래 Packet에 바깥 Header를 추가하는 것을 **Encapsulation**이라고 한다.

```text
원래 Pod Packet
src: Pod A IP
dst: Pod B IP

Overlay에서 Node 사이를 지날 때
Outer src/dst: Node A IP → Node B IP
└─ Inner src/dst: Pod A IP → Pod B IP
```

| 방식 | 장점 | 고려사항 |
|---|---|---|
| Non-overlay | 추가 Header가 없어 구조와 성능이 단순함 | Underlay가 Pod CIDR의 경로를 알아야 함 |
| Overlay | 기존 Underlay를 크게 변경하지 않아도 됨 | Encapsulation에 따른 Header와 MTU 고려 필요 |

### NetworkPolicy와 ACL

Kubernetes `NetworkPolicy`는 어떤 Pod가 다른 Pod나 외부 대상으로 통신할 수 있는지를 선언하는 API다. 기본 Kubernetes API는 주로 L3/L4의 IP, Pod·Namespace Selector, Protocol, Port를 기준으로 Ingress와 Egress 허용 규칙을 표현한다.

```text
NetworkPolicy 예시

frontend Label을 가진 Pod만
database Pod의 TCP 5432 Port로 접근 허용
```

NetworkPolicy Object를 생성하는 것만으로 Packet이 차단되지는 않는다. Network Plugin이 이를 감시하고 Linux의 iptables, nftables, eBPF 같은 실제 Data Plane 규칙으로 변환해야 한다.

**ACL(Access Control List)**은 이렇게 Data Plane에 적용되는 허용·차단 규칙을 넓게 부르는 용어다. 예를 들어 Calico의 Felix는 NetworkPolicy를 읽은 결과를 Node의 ACL로 구성한다.

Flannel의 `flanneld`는 자체적으로 NetworkPolicy를 적용하지 않는다. Policy가 필요하면 다음과 같이 별도 구현을 함께 사용한다.

- Flannel Helm Chart의 Kubernetes SIGs Network Policy Controller 사용
- Flannel Networking과 Calico Policy를 조합한 Canal 구성
- CNI Chaining을 이용해 별도 Policy 구현 추가

---

## 8. Flannel Overview

Flannel은 Kubernetes를 위한 비교적 단순한 Layer 3 Network Fabric을 제공하는 프로젝트다.

각 Node에서 `flanneld`가 실행되며, 전체 Pod Network 대역에서 해당 Node가 사용할 **Subnet Lease**를 확보한다. 여기서 Subnet Lease는 Pod IP 하나의 임대가 아니라, 특정 Node가 사용할 Pod CIDR과 그 Node로 도달하기 위한 정보를 기록한 할당 단위다.

예를 들어 전체 Pod CIDR이 `10.244.0.0/16`이라면 다음처럼 나눌 수 있다.

```text
Node A → 10.244.1.0/24
Node B → 10.244.2.0/24
```

Flannel은 Kubernetes API 또는 etcd에 이 정보를 저장하고 다른 Node의 Subnet 정보를 확인한다. 개별 Pod IP는 기본적으로 `host-local` IPAM Plugin이 해당 Node의 Subnet 안에서 할당한다.

### 구성요소와 Pod 생성 흐름

```text
Kubernetes API
  ├─ Node A PodCIDR: 10.244.1.0/24
  └─ Node B PodCIDR: 10.244.2.0/24
            │
            ▼
Node별 flanneld
  ├─ Subnet Lease 확인
  ├─ Backend 장치와 Route 구성
  └─ /run/flannel/subnet.env 기록
            │
            ▼
Flannel CNI Plugin
  └─ bridge Plugin에 설정 위임
       ├─ veth와 cni0 Bridge 연결
       └─ host-local IPAM으로 Pod IP 할당
```

`flanneld`는 계속 실행되면서 Node 간 Network를 관리한다. 반면 Flannel CNI Plugin은 Pod 생성·삭제 시 실행되고, `subnet.env`의 Subnet과 MTU를 이용해 기본적으로 `bridge` Plugin과 `host-local` IPAM을 호출한다.

### Node 간 Packet 전달

Flannel은 Backend에 따라 Node 사이의 전달 방식이 달라진다.

다음 환경에서 Pod A가 Pod B로 Packet을 보낸다고 가정한다.

```text
Node A IP: 192.168.0.11        Node B IP: 192.168.0.12
Pod A: 10.244.1.5              Pod B: 10.244.2.8
```

#### VXLAN Backend

```text
Pod A
  → veth → cni0
  → flannel.1에서 VXLAN Encapsulation
      Outer: Node A IP → Node B IP
      Inner: Pod A IP  → Pod B IP
  → Underlay Network
  → Node B의 flannel.1에서 Decapsulation
  → cni0 → veth → Pod B
```

Underlay는 Node IP 사이만 전달할 수 있으면 되고 Pod CIDR을 알 필요가 없다. 대신 VXLAN Header가 추가되므로 Pod Interface의 MTU를 더 작게 잡아야 한다.

#### host-gw Backend

`host-gw`는 각 Node를 다른 Pod Subnet으로 가는 Gateway로 사용한다. Node A에는 다음과 비슷한 Route가 만들어진다.

```text
10.244.2.0/24 via 192.168.0.12 dev eth0
```

```text
Pod A
  → veth → cni0
  → Node A가 Route 조회
  → 다음 Hop인 Node B IP로 원본 Pod Packet 전달
  → Node B → cni0 → veth → Pod B
```

별도 Tunnel Header가 없다는 장점이 있지만 다음 Underlay 조건이 필요하다.

- Node들이 서로 직접 Layer 2로 연결되어 Node IP를 다음 Hop으로 사용할 수 있어야 한다.
- 중간 Router가 있다면 해당 Router가 Pod CIDR Route를 알아야 하므로 단순한 `host-gw` 전제에서 벗어난다.
- 방화벽과 보안 규칙이 Node 간 Pod Traffic을 허용해야 한다.

공식 Flannel 문서는 일반적인 환경에는 VXLAN을 권장하고, 이 조건을 만족하며 캡슐화 비용을 줄이고 싶을 때 `host-gw`를 권장한다.

### 특징

- Pod 간 기본 연결에 집중한 비교적 단순한 구조
- Node별 Pod Subnet과 Backend를 이용한 Node 간 통신
- 기본 Backend는 VXLAN

### 장점과 고려사항

| 장점 | 고려사항 |
|---|---|
| 구성요소가 비교적 단순해 기본 Pod Network를 이해하고 운영하기 쉬움 | `flanneld` 자체에는 NetworkPolicy 적용 기능이 없음 |
| VXLAN은 Underlay가 Pod CIDR을 몰라도 동작 | Security와 Observability 기능은 다른 도구와 조합해야 함 |
| `host-gw`는 Encapsulation 없이 전달 가능 | `host-gw`는 Node 사이의 Network 구조에 제약이 있음 |

---

## 9. Calico Overview

Calico는 Pod Networking과 NetworkPolicy를 함께 제공하는 Network Solution이다. Routing 중심의 구조를 가지며 Overlay와 Non-overlay 구성을 모두 지원한다.

주요 구성 요소는 다음과 같다.

| 구성 요소 | 역할 |
|---|---|
| Calico CNI Plugin | Pod의 Network Namespace와 Host를 veth로 연결 |
| Calico IPAM | IP Pool을 Node별 Block으로 나누고 Pod IP 할당·회수 |
| Felix | 각 Node에서 Interface, Route, NetworkPolicy의 ACL을 Data Plane에 반영 |
| BIRD | BGP Mode에서 다른 Node나 외부 Router와 Pod Route 교환 |
| confd | Calico 상태를 바탕으로 BIRD 설정 생성 |
| kube-controllers | Kubernetes의 Pod, Namespace, Policy, Node 상태를 Calico Data Model과 동기화 |

### Pod 생성과 제어 흐름

```text
Pod 생성
  → Container Runtime이 Calico CNI 실행
  → Calico IPAM이 IP Pool에서 Pod IP 할당
  → Calico CNI가 Pod eth0 ↔ Host cali* veth 연결
  → Workload Endpoint 정보가 Calico Data Model에 기록

Kubernetes API / Calico CRD
  ├─ kube-controllers: Kubernetes 상태 동기화
  │
  └─ Node별 Felix
       ├─ Local Pod Route와 Interface 구성
       ├─ NetworkPolicy를 ACL 규칙으로 변환
       └─ 선택한 Data Plane에 규칙 적용
            ├─ 표준 Linux: iptables 또는 nftables
            └─ eBPF Mode: eBPF Program과 Map

BGP Mode일 때
Calico 상태 → confd → BIRD → 다른 Node 또는 Router와 Route 교환
```

Calico에서 Cluster 전체 상태를 다루는 `kube-controllers`와 각 Node의 실제 Network를 구성하는 Felix는 역할이 다르다. Packet은 `kube-controllers`를 통과하지 않고 Felix가 미리 구성한 Kernel Data Plane을 지난다.

### Node 간 Packet 전달

Calico는 환경에 따라 다음 방식을 선택할 수 있다.

#### Non-overlay와 BGP

```text
Pod A
  → cali* veth
  → Node A에서 Egress Policy 검사
  → Linux Route 조회
  → 원래 Pod IP Packet을 Node B 방향으로 전달
  → Node B에서 Ingress Policy 검사
  → 목적지 cali* veth → Pod B
```

BIRD가 BGP로 `Pod CIDR → Node` Route를 교환하거나 Underlay Router가 Pod CIDR을 알고 있어야 한다. Encapsulation이 없으므로 Packet에 추가 Tunnel Header가 붙지 않는다.

#### VXLAN과 IP-in-IP Overlay

- **VXLAN**은 원래 Ethernet Frame을 UDP Packet 안에 넣는다.
- **IP-in-IP**는 원래 IPv4 Packet 전체를 새로운 IPv4 Packet 안에 넣는다. 구조가 단순하지만 IPv4 Traffic만 캡슐화할 수 있다.

```text
Inner Packet: Pod A IP → Pod B IP
       ↓ Encapsulation
Outer Packet: Node A IP → Node B IP
       ↓ Underlay를 통해 전달
Node B에서 Decapsulation 후 Pod B로 Routing
```

#### Cross-subnet Mode

Cross-subnet은 두 방식을 섞는다.

```text
Node가 같은 Underlay Subnet에 있음
  → Encapsulation 없이 직접 Routing

Node가 서로 다른 Underlay Subnet에 있음
  → VXLAN 또는 IP-in-IP로 Encapsulation
```

같은 Subnet에서는 Tunnel 비용을 줄이고, Subnet 경계를 넘을 때만 Underlay의 Route 제약을 피할 수 있다.

따라서 Calico를 단순히 “BGP만 사용하는 CNI” 또는 “Overlay CNI”라고 정의하면 정확하지 않다. 설치 환경과 설정에 따라 Data Path가 달라질 수 있다.

### 특징

- Routing과 NetworkPolicy를 함께 제공
- 자체 IPAM으로 IP Pool과 Node별 IP Block 관리 가능
- BGP 기반 Non-overlay와 VXLAN/IP-in-IP Overlay 지원
- 표준 Linux, nftables, eBPF 등 설치 설정에 따른 Data Plane 선택 가능
- Kubernetes NetworkPolicy뿐 아니라 Calico 자체 Policy 기능도 제공

### 장점과 고려사항

| 장점 | 고려사항 |
|---|---|
| Overlay와 Non-overlay를 환경에 맞게 선택 가능 | 선택할 수 있는 Mode와 구성요소가 많아 설정이 복잡해질 수 있음 |
| Kubernetes NetworkPolicy와 확장된 Calico Policy 제공 | BGP 연동 시 Routing 지식과 Underlay 협의가 필요함 |
| 자체 IPAM으로 IP Pool과 Node별 Block을 세밀하게 관리 | Overlay 사용 시 Header와 MTU Overhead가 생김 |
| 기존 Linux Data Plane부터 eBPF까지 선택 가능 | 실제 Packet 경로가 선택한 Data Plane에 따라 달라져 장애 분석 시 설정 확인이 중요함 |

---

## 10. Cilium Overview

Cilium은 Linux Kernel의 eBPF를 중심으로 Networking, Security, Observability를 제공하는 Network Solution이다.

**eBPF**는 검증된 작은 Program을 Linux Kernel의 여러 Hook에 연결해 Packet이나 System Call을 처리할 수 있게 하는 기술이다. Cilium은 Pod veth의 Traffic Control Hook, Socket Hook, XDP 등 필요한 위치에 eBPF Program을 연결한다.

- **eBPF Program**: Packet 허용·차단, Forwarding, Service Load Balancing 같은 동작 수행
- **eBPF Map**: Endpoint, Identity, Policy, Service Backend, Connection Tracking 같은 상태 저장

각 Node의 `cilium-agent`가 Kubernetes 상태를 확인하고 이 Program과 Map을 갱신한다.

| 구성 요소 | 역할 |
|---|---|
| `cilium-cni` | Pod 생성·삭제 시 같은 Node의 Agent에 Network 연결 요청 |
| `cilium-agent` | Kubernetes Event를 감시하고 Node의 eBPF Program과 Map 관리 |
| Cilium Operator | IPAM과 Node 정보처럼 Cluster 단위로 조정할 작업 수행 |
| Hubble Server | 각 Node Agent 안에서 eBPF Flow Event 수집 |
| Hubble Relay | 여러 Node의 Hubble Server를 연결해 Cluster 전체 Flow 제공 |

### Pod 생성과 제어 흐름

```text
Kubernetes API
  ├─ Pod / Node
  ├─ Service / EndpointSlice
  ├─ Kubernetes NetworkPolicy
  └─ CiliumNetworkPolicy
          │
          ├───────────────┐
          ▼               ▼
 Cilium Operator     Node별 cilium-agent
 Cluster 단위 작업    ├─ Local Endpoint와 Identity 관리
 IPAM 조정 등         ├─ Policy를 eBPF 규칙으로 변환
                     ├─ Service와 Backend를 LB Map에 기록
                     └─ eBPF Program을 Kernel Hook에 연결
                              │
                              ▼
                     Linux Kernel eBPF Data Plane

Pod 생성
  → Container Runtime이 cilium-cni 실행
  → cilium-cni가 Local cilium-agent에 요청
  → veth와 Endpoint 생성
  → Identity, Policy, Route와 eBPF 상태 구성
```

Operator와 Agent는 대체 관계가 아니다. Operator는 Cluster 전체에서 한 번 조정할 작업을 담당하고, Agent는 각 Node의 실제 Data Plane을 구성한다. 실제 Packet은 Operator를 통과하지 않는다.

### Node 간 Packet 전달

Cilium도 환경에 따라 여러 Network Mode를 지원한다.

#### Overlay Mode

- **VXLAN**: Ethernet Frame을 UDP Packet으로 캡슐화하는 방식
- **Geneve**: VXLAN과 비슷한 UDP 기반 Overlay Protocol이지만 확장 가능한 Option Header에 추가 Metadata를 담기 쉽게 설계된 방식

```text
Pod A
  → veth의 eBPF Program에서 Policy와 목적지 확인
  → VXLAN 또는 Geneve Encapsulation
  → Node A IP → Node B IP로 Underlay 통과
  → Node B에서 Decapsulation
  → eBPF Policy 확인 → Pod B
```

#### Native Routing Mode

`Native`라는 이름은 Cilium이 만든 Overlay Tunnel을 사용하지 않고 Linux의 기본 Routing 기능과 Underlay를 그대로 사용한다는 의미다.

```text
Pod A
  → eBPF Policy와 Forwarding 처리
  → Linux Routing Table
  → Underlay가 Pod B IP 또는 Pod CIDR을 Node B로 Routing
  → Node B의 eBPF Policy 확인
  → Pod B
```

따라서 Underlay가 Pod CIDR을 Routing할 수 있어야 한다. Cilium은 환경에 따라 L2 Neighbor Discovery나 BGP를 이용해 필요한 Route를 알릴 수 있다.

### Service를 처리하는 흐름

kube-proxy는 Service와 EndpointSlice를 감시해 Kernel에 전달 규칙을 만드는 컴포넌트다. kube-proxy가 모든 Packet을 직접 중계하는 것은 아니다.

Cilium의 kube-proxy Replacement를 활성화하면 이 역할을 `cilium-agent`와 eBPF Data Plane이 대신한다.

```text
Kubernetes Service / EndpointSlice 변경
  → 각 Node의 cilium-agent가 확인
  → Service IP와 Backend Pod를 eBPF LB Map에 기록

Client Pod가 ClusterIP로 연결
  → Socket 또는 veth의 eBPF Program 실행
  → LB Map에서 Service와 Backend 조회
  → Backend 하나 선택
  → 연결 대상을 Backend Pod IP와 Port로 변환
  → Cilium Pod Network를 통해 Backend로 전달
```

Socket Load Balancing에서는 애플리케이션의 `connect()` 시점에 Service 주소를 Backend 주소로 바꿀 수 있다. Socket Hook을 사용할 수 없는 경로에서는 veth의 Traffic Control Hook에서 Service 조회와 변환을 수행할 수 있다.

### 특징

- eBPF 기반 Data Plane
- L3/L4뿐 아니라 선택적으로 L7 Policy와 가시성 제공
- Hubble을 통한 Flow 관찰 가능
- 설정에 따라 eBPF로 Kubernetes Service를 처리해 kube-proxy를 대체할 수 있음

Policy 범위를 예로 들면 다음과 같다.

| Layer | Policy 예시 |
|---|---|
| L3 | `frontend` Identity를 가진 Pod만 `api` Pod에 접근 허용 |
| L4 | 허용된 Pod 중 TCP 8080 Traffic만 허용 |
| L7 | HTTP `GET /public`은 허용하고 다른 Method나 Path는 차단 |

Kubernetes 기본 NetworkPolicy는 주로 L3/L4 범위를 다룬다. Cilium의 L7 Policy는 CiliumNetworkPolicy와 Proxy 연동을 통해 HTTP Method나 Path 같은 애플리케이션 정보를 검사한다.

`Cilium을 설치하면 항상 kube-proxy가 사라진다`는 의미는 아니다. kube-proxy 대체 여부는 Cilium 설치 설정에 따라 달라진다.  

### 장점과 고려사항

| 장점 | 고려사항 |
|---|---|
| Network, Policy, Service 처리와 관찰 기능을 eBPF 중심으로 통합 | 지원하는 Linux Kernel Version과 기능을 확인해야 함 |
| Hubble로 허용·차단 Flow와 Service 통신을 관찰 가능 | eBPF Hook과 Map을 이해해야 깊은 장애 분석이 가능함 |
| 선택적으로 kube-proxy를 대체해 Service 처리를 통합 | 기능과 설정 범위가 넓어 초기 학습과 운영 복잡도가 커질 수 있음 |
| Identity 기반 Policy와 L7 Policy 지원 | L7 처리는 Proxy 연동이 추가되므로 L3/L4보다 처리 경로가 복잡함 |

---

## 11. Flannel, Calico, Cilium 비교

세 Solution 모두 Kubernetes Pod Network를 구현할 수 있지만, 중점을 두는 범위와 Data Plane이 다르다.


| 구분            | Flannel                    | Calico                                              | Cilium                              |
| ------------- | -------------------------- | --------------------------------------------------- | ----------------------------------- |
| 주된 초점         | 기본 Pod Connectivity        | Networking과 NetworkPolicy                           | Networking, Security, Observability |
| Node Agent    | `flanneld`                 | Felix, 필요 시 BIRD                                    | `cilium-agent`                      |
| Node 간 통신     | VXLAN, host-gw 등           | Non-overlay 또는 Overlay                              | Native Routing 또는 Overlay           |
| Packet 처리     | Linux Routing과 선택한 Backend | 표준 Linux, nftables 또는 eBPF                          | eBPF 중심                             |
| IPAM          | 주로 `host-local`과 조합        | Calico IPAM 제공                                      | 여러 Cilium IPAM Mode 제공              |
| NetworkPolicy | `flanneld` 외 별도 구현 필요      | 지원                                                  | 지원                                  |
| Service 처리    | 일반적으로 kube-proxy와 함께 사용    | 표준 Data Plane은 kube-proxy와 함께 사용하며 eBPF Mode는 대체 가능 | 선택적으로 kube-proxy 대체 가능              |


이 표는 대표적인 구성을 단순화한 것이다. 실제 기능은 Version, 설치 방식, Cloud 환경, 선택한 Data Plane에 따라 달라질 수 있다.  

### Cluster Controller와 Node Agent 비교

Calico만 Kubernetes API와 Controller를 사용하고 Cilium만 Operator-Agent 구조를 사용하는 것은 아니다. 두 Solution 모두 Kubernetes API를 상태의 출처로 사용하고, Cluster 단위 구성요소와 Node 단위 Agent를 나눠 둔다. 이름과 세부 책임이 다를 뿐 기본 구조는 비슷하다.

| 범위 | Flannel | Calico | Cilium |
|---|---|---|---|
| Kubernetes 상태 확인 | 각 `flanneld`가 Node와 Subnet 정보 확인 | `kube-controllers`와 Felix가 필요한 상태 확인 | Operator와 `cilium-agent`가 필요한 상태 확인 |
| Cluster 단위 처리 | 별도 중앙 Controller가 적은 단순한 구조 | `kube-controllers`가 Kubernetes 상태를 Calico Data Model과 동기화 | Operator가 IPAM 등 Cluster 단위 작업 조정 |
| Node 단위 처리 | `flanneld`가 Backend와 Route 관리 | Felix가 Route와 Policy Data Plane 관리 | `cilium-agent`가 eBPF Data Plane 관리 |

```text
Calico
Kubernetes API → kube-controllers → Calico 상태 → Node별 Felix → Kernel Data Plane

Cilium
Kubernetes API → Operator: Cluster 단위 작업
               └→ Node별 cilium-agent → eBPF Data Plane
```

Calico의 Felix도 Agent이고 Cilium Agent도 Kubernetes API 상태를 사용한다. `Controller`와 `Operator`라는 이름만 보고 서로 완전히 다른 구조라고 이해하면 안 된다.

### 가장 큰 차이

```text
Flannel
  → Pod가 Node를 넘어 통신할 기본 Network를 단순하게 제공

Calico
  → Routing과 NetworkPolicy를 중심으로 다양한 Network 구성을 제공

Cilium
  → eBPF를 중심으로 Network, Policy, Service, Observability 범위를 확장
```

무조건 기능이 많은 Solution이 더 적합한 것은 아니다. Cluster 환경, 운영 복잡도, NetworkPolicy 요구사항, 관찰 기능, 기존 Underlay Network와의 연동 방식을 함께 고려해야 한다.

---

## 12. 현재 Cluster에서 Calico가 담당하는 위치

현재 학습 Cluster가 Calico를 Primary CNI로 사용한다면 각 요소의 관계는 다음과 같이 볼 수 있다.

```text
Pod 생성
  → Container Runtime이 Calico CNI 실행
  → Calico IPAM이 Pod IP 할당
  → Calico CNI가 Pod Interface 연결
  → Felix가 Node의 Route와 Policy 구성
  → BGP 또는 Overlay 설정에 따라 다른 Node로 전달
```

이때 Calico가 설치되어 있다는 사실만으로 실제 통신 방식이 하나로 결정되지는 않는다. 다음 설정을 함께 확인해야 한다.

- Pod IP를 어느 IP Pool에서 할당하는가?
- VXLAN, IP-in-IP, BGP 중 어떤 방식이 활성화되어 있는가?
- NetworkPolicy는 어떤 Data Plane에서 적용되는가?
- Service 처리는 kube-proxy가 담당하는가, 다른 Data Plane이 담당하는가?

Week 2와 연결하면 역할은 다음과 같이 구분된다.


| 대상                | 담당 역할                                    |
| ----------------- | ---------------------------------------- |
| Calico CNI / IPAM | Pod Interface와 IP 구성                     |
| Felix와 Routing 구성 | Pod IP까지 실제 Packet 전달                    |
| NetworkPolicy 기능  | 허용된 Packet인지 검사                          |
| kube-proxy        | 일반적인 구성에서 Service IP를 Backend Pod IP로 연결 |


---

## 13. 학습하면서 생긴 질문

### Q1. CNI Plugin은 주로 Pod 생성·삭제 시 실행되는데, 실행이 끝난 뒤에도 다른 Node와 계속 통신할 수 있는 이유는 무엇일까?

CNI Plugin이 종료되어도 Plugin이 만들어 놓은 Interface와 Route, Kernel의 Data Path는 남아 있다. 또한 Flannel의 `flanneld`, Calico의 Felix, Cilium의 `cilium-agent` 같은 Node Agent가 Cluster 상태 변화를 계속 확인하며 Node 간 경로나 Policy를 갱신한다.

즉, **CNI 호출은 Pod를 Network에 연결하는 생명주기 작업**이고, **상시 실행되는 Agent와 Kernel Data Plane은 연결 상태를 유지하는 역할**을 한다.

### Q2. 모두 같은 CNI Specification을 따르는데 Flannel, Calico, Cilium의 구조가 다른 이유는 무엇일까?

CNI는 Runtime이 Plugin을 호출하고 결과를 받는 경계만 표준화한다. Node 간 Routing, Overlay Protocol, Policy, Service 처리, 관찰 기능까지 하나의 방식으로 제한하지 않는다.

따라서 각 Solution은 같은 CNI Interface 뒤에서 VXLAN, BGP, Linux Routing, eBPF 등 서로 다른 기술을 선택할 수 있다.

---

## 14. 핵심 용어 정리


| 용어               | 설명                                                          |
| ---------------- | ----------------------------------------------------------- |
| CNI              | Runtime과 Network Plugin 사이의 실행 방법과 결과 형식을 정의한 명세            |
| CNI Plugin       | CNI 명세에 따라 실제 Network 구성을 수행하는 프로그램                         |
| Network Solution | CNI Plugin과 Node Agent 등을 포함해 Pod Network를 구현하는 전체 Solution |
| Primary CNI      | Cluster의 기본 Pod Network 연결을 담당하는 CNI 기반 Solution            |
| IPAM             | IP 주소의 할당, 중복 방지, 회수를 관리하는 기능                               |
| Subnet Lease     | Flannel이 특정 Node에 배정한 Pod Subnet과 해당 Node의 도달 정보              |
| Plugin Chaining  | 여러 CNI Plugin을 순서대로 실행해 기능을 조합하는 방식                         |
| Control Plane    | 원하는 Network 상태를 계산하고 Data Plane에 반영하는 영역                    |
| Data Plane       | 실제 Packet을 전달하거나 차단하는 경로                                    |
| NetworkPolicy    | Pod의 Ingress와 Egress 허용 범위를 선언하는 Kubernetes API              |
| ACL              | Source, Destination, Protocol, Port 등에 따른 접근 허용·차단 규칙          |
| Overlay          | 기존 Network 위에서 Packet을 캡슐화해 논리 Network를 구성하는 방식             |
| Non-overlay      | 별도 Tunnel 없이 Underlay Route를 이용해 Pod Packet을 전달하는 방식        |
| Cross-subnet     | 같은 Subnet에서는 직접 Routing하고 다른 Subnet 사이에서만 캡슐화하는 방식        |
| VXLAN            | Ethernet Frame을 UDP Packet 안에 넣는 Overlay Protocol             |
| IP-in-IP         | IPv4 Packet을 새로운 IPv4 Packet 안에 넣는 Tunnel 방식                   |
| Geneve           | 확장 가능한 Option Header를 지원하는 UDP 기반 Overlay Protocol            |
| BGP              | Network 장비나 Node 사이에서 Route 정보를 교환하는 Protocol               |
| eBPF             | Linux Kernel에서 Network 처리와 관찰 등의 Program을 실행할 수 있는 기술       |
| eBPF Map         | eBPF Program과 User Space가 Network 상태를 공유하는 Key-Value 저장소      |


---

## 15. 참고 자료

- [Kubernetes Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
- [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Kubernetes Services, Load Balancing, and Networking](https://kubernetes.io/docs/concepts/services-networking/)
- [CNI Specification](https://github.com/containernetworking/cni/blob/main/SPEC.md)
- [CNI Reference Plugins](https://github.com/containernetworking/plugins)
- [Flannel 공식 문서](https://github.com/flannel-io/flannel)
- [Flannel CNI Plugin](https://github.com/flannel-io/cni-plugin)
- [Flannel Backends](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md)
- [Calico Component Architecture](https://docs.tigera.io/calico/latest/reference/architecture/overview)
- [Calico Data Path](https://docs.tigera.io/calico/latest/reference/architecture/data-path)
- [Calico Networking Options](https://docs.tigera.io/calico/latest/networking/determine-best-networking)
- [Calico eBPF Data Plane](https://docs.tigera.io/calico/latest/operations/ebpf/enabling-ebpf)
- [Cilium Introduction](https://docs.cilium.io/en/stable/overview/intro/)
- [Cilium Component Overview](https://docs.cilium.io/en/stable/overview/component-overview/)
- [Cilium Kubernetes Without kube-proxy](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)
- [Cilium eBPF Maps](https://docs.cilium.io/en/stable/network/ebpf/maps/)
- [Cilium Layer 7 Policies](https://docs.cilium.io/en/stable/security/policy/layer7/)
