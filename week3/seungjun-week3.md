# Week 3. CNI Overview & Cloud Native Networking Landscape

## 1. 이번 주 학습 목표

Week 2에서는 Pod가 생성될 때 `kubelet → CRI → Container Runtime → CNI`를 거쳐 Network가 구성되는 흐름을 살펴봤다.

이번 주에는 CNI가 정확히 어떤 범위까지 정의하는지 알아보고, Kubernetes의 Pod Network를 실제로 구현하는 대표적인 Network Solution인 **Flannel, Calico, Cilium**의 구조와 차이를 비교한다.

- CNI가 표준화하는 범위 이해
- CNI Plugin과 IPAM의 역할 이해
- Primary CNI의 의미 이해
- Overlay / Routing 등 Node 간 Network 구성 방식 이해
- Flannel / Calico / Cilium의 기본 Architecture 비교

---

## 2. CNI(Container Network Interface)

### CNI란?

**CNI(Container Network Interface)** 는 Container Runtime이 Container 또는 Pod를 Network에 연결할 때 사용하는 **표준 Interface**이다.

Container Runtime마다 Network Solution을 호출하는 방법이 다르면 각각 별도의 연동 방식을 구현해야 한다.

CNI는 Runtime과 Network Plugin 사이에 공통 규약을 정의하여 이 문제를 해결한다.

```text
Container Runtime
        │
        │ CNI 규약
        ▼
   CNI Plugin
        │
        ▼
Pod Network 구성
```

Container Runtime은 CNI 규약에 맞춰 Plugin을 호출하고, CNI Plugin은 필요한 Network 구성을 수행한다.

CNI는 Kubernetes 전용 기술이 아니라 Container Network를 위한 CNCF 표준이다.

---

## 3. CNI가 정의하는 범위

CNI는 **Network를 어떤 기술로 구현할지**를 정하는 규격이 아니다.

CNI가 정의하는 핵심은 Runtime이 Network Plugin을 **어떤 방식으로 호출하고 어떤 정보를 주고받을 것인가**이다.

### 주요 CNI Command

| Command | 역할 |
| --- | --- |
| `ADD` | Container를 Network에 연결 |
| `DEL` | Network 연결 제거 및 자원 정리 |
| `CHECK` | Network가 정상적으로 구성되어 있는지 확인 |
| `VERSION` | Plugin이 지원하는 CNI Version 확인 |

Runtime은 Plugin을 실행할 때 Network Namespace, Interface 이름, Container ID 등의 정보를 전달한다.

```text
Container Runtime
        │
        │ ADD
        │ Network Namespace
        │ Interface Name
        │ Network Config
        ▼
   CNI Plugin
        │
        ├─ Interface 구성
        ├─ IP 설정
        └─ Route 설정
        │
        ▼
       Result
```

CNI Plugin은 일반적으로 Pod 생성 시 실행되고 작업이 끝나면 종료되는 **실행 파일**이다.

노드에는 대표적으로 다음 위치에 CNI 관련 파일이 존재한다.

```text
/etc/cni/net.d/
→ 어떤 CNI Plugin을 사용할지 정의한 설정

/opt/cni/bin/
→ Runtime이 실제로 실행하는 CNI Plugin
```

---

## 4. CNI와 Network Solution

여기서 CNI라는 용어를 구분해서 볼 필요가 있다.

### CNI Specification

Runtime과 Plugin 사이의 호출 방법을 정의한 표준이다.

### CNI Plugin

CNI Specification을 구현한 실행 파일이다.

```text
bridge
host-local
calico
cilium-cni
```

### CNI 기반 Network Solution

실제 Kubernetes Cluster Network 전체를 구성하는 Solution이다.

예를 들면:

```text
Flannel
Calico
Cilium
```

이들은 단순한 CNI Plugin 하나가 아니라 일반적으로 다음과 같은 여러 Component로 구성된다.

```text
CNI Network Solution

├─ CNI Plugin
├─ Node Agent
├─ Controller
├─ IPAM
└─ Routing / Policy 등의 Network 기능
```

CNI Plugin은 Pod가 생성되는 시점의 Network 설정을 수행한다.

반면 Node Agent나 Controller는 계속 실행되면서 Node 추가, Routing 변경, NetworkPolicy 변경 등의 Cluster Network 상태를 관리한다.

---

## 5. CNI Plugin과 IPAM

### IPAM이란?

**IPAM(IP Address Management)**은 Network에서 사용할 **IP Address를 할당하고, 사용 상태를 관리하며, 더 이상 사용하지 않는 IP를 회수하는 기능**이다.

Pod마다 고유한 IP가 필요하기 때문에 다음과 같은 상태 관리가 필요하다.

```text
Pod A → 10.244.1.10
Pod B → 10.244.1.11
Pod C → 10.244.1.12
```

어떤 IP가 이미 사용 중인지 관리하지 않으면 두 Pod에 동일한 IP가 할당될 수 있다.

또 삭제된 Pod의 IP를 회수하지 않으면 사용할 수 있는 IP가 점점 줄어든다.

따라서 IPAM은 다음 역할을 담당한다.

- 사용 가능한 IP 선택
- IP 중복 할당 방지
- IP 할당 상태 저장
- Pod 삭제 시 IP 회수
- IP 재사용

---

### CNI Plugin과 IPAM의 관계

Network Interface를 구성하는 Plugin과 IP를 관리하는 기능을 분리해서 사용할 수 있다.

```text
CNI Main Plugin
      │
      │ IP 요청
      ▼
    IPAM
      │
      │ 사용 가능한 IP 반환
      ▼
CNI Main Plugin
      │
      ├─ Interface 생성
      ├─ IP 설정
      └─ Route 설정
```

대표적인 IPAM Plugin에는 다음과 같은 것들이 있다.

| IPAM | 특징 |
| --- | --- |
| `host-local` | Node 내부에서 IP 할당 상태 관리 |
| `dhcp` | DHCP Server를 이용해 IP 할당 |
| `calico-ipam` | Calico의 IPPool을 이용해 IP 관리 |

Network Solution에 따라 자체적인 IPAM 방식을 사용할 수도 있다.

---

## 6. Primary CNI

### Primary CNI란?

**Primary CNI**는 Kubernetes Cluster에서 Pod의 **기본 Network Interface와 Pod Network를 구성하는 Network Solution**을 의미한다.

Pod를 생성하면 일반적으로 `eth0`이라는 기본 Interface가 만들어진다.

```text
Pod
└─ eth0
     ▲
     │
Primary CNI
```

대표적인 Primary CNI Solution은 다음과 같다.

- Flannel
- Calico
- Cilium
- AWS VPC CNI
- Azure CNI

Primary CNI가 Pod Network를 구성하면 Pod는 자신의 IP를 이용해 다른 Pod와 통신할 수 있다.

---

## 7. Node 간 Pod Network

하나의 Node 안에서도 Pod마다 별도의 Network Namespace와 IP가 필요하기 때문에 CNI가 필요하다.

Cluster가 여러 Node로 구성되면 추가적인 문제가 발생한다.

```text
Node A                        Node B

Pod A                         Pod B
10.244.1.10                   10.244.2.20
   │                             │
   └──────── 어떻게 통신? ────────┘
```

Node A는 `10.244.2.20`이라는 Pod가 Node B에 있다는 사실을 알아야 하고, Packet을 Node B까지 전달할 방법도 필요하다.

Primary CNI Solution들은 이 문제를 서로 다른 방식으로 해결한다.

이때 자주 등장하는 개념이 **Overlay Network, VXLAN, Routing, BGP**이다.

---

## 8. Overlay Network와 VXLAN

### Overlay Network

**Overlay Network**는 기존 Network 위에 별도의 가상 Network를 구성하는 방식이다.

기존 물리 Network는 Node IP끼리 통신할 수 있으면 되고, Pod Network는 그 위에 별도로 구성한다.

```text
Pod Network
─────────────────────────
        Overlay

Node A ───────────── Node B
─────────────────────────
       실제 Network
```

Overlay Network를 구현할 때 대표적으로 사용하는 기술 중 하나가 **VXLAN**이다.

---

### VXLAN

**VXLAN(Virtual Extensible LAN)** 은 원래 Packet을 다른 Packet 안에 한 번 더 넣어 전달하는 **Tunnel 기술**이다.

예를 들어 Pod A가 다른 Node의 Pod B로 Packet을 보낸다고 가정한다.

```text
Pod A
10.244.1.10
   │
   ▼

원래 Packet
10.244.1.10 → 10.244.2.20

   │
   │ VXLAN Encapsulation
   ▼

Node A IP → Node B IP
┌─────────────────────────┐
│ 기존 Pod Packet          │
│ 10.244.1.10 → 10.244.2.20│
└─────────────────────────┘

   │
   ▼

Node B
VXLAN Packet 해제

   │
   ▼
Pod B
10.244.2.20
```

실제 Network는 Pod IP를 알 필요 없이 Node A와 Node B의 IP만 알면 된다.

이 때문에 기존 Network 환경을 크게 변경하지 않고 Kubernetes Pod Network를 구성하기 쉽다는 특징이 있다.

---

## 9. Routing과 BGP

Node 간 Pod Network를 구성하는 또 다른 방법은 Packet을 감싸지 않고 **Routing 정보를 이용해 직접 전달하는 방식**이다.

### Routing

Routing은 목적지 IP를 보고 **Packet을 어느 경로로 전달해야 하는지 결정하는 과정**이다.

Node A에 다음과 같은 Routing 정보가 있다고 가정한다.

```text
10.244.2.0/24
→ Node B로 전달
```

그러면 `10.244.2.20`으로 향하는 Packet을 Node B로 전달할 수 있다.

---

### BGP

**BGP(Border Gateway Protocol)**는 Network 장비들이 자신이 알고 있는 **IP Network 경로를 서로 알려주는 Routing Protocol**이다.

Kubernetes Network에서는 Node가 자신이 가지고 있는 Pod Network 정보를 다른 Node에 알려주는 데 사용할 수 있다.

```text
Node A
"10.244.1.0/24는 나에게 보내"

Node B
"10.244.2.0/24는 나에게 보내"
```

이 정보를 서로 공유하면:

```text
Node A Routing Table

10.244.2.0/24
→ Node B
```

와 같은 경로를 구성할 수 있다.

따라서 Packet을 VXLAN으로 한 번 더 감싸지 않고 기존 Routing을 이용해 다른 Node로 전달할 수도 있다.

---

## 10. Flannel

Flannel은 **Pod 간 Network 연결을 비교적 단순하게 제공하는 Network Solution**이다.

대표적으로 VXLAN 기반 Overlay Network를 사용할 수 있다.

```text
Pod
 │
veth
 │
cni0
 │
flannel.1
 │
VXLAN
 │
다른 Node
```

### 주요 Component

#### flanneld

각 Node에서 실행되는 Agent이다.

다른 Node의 Network 정보를 확인하고 VXLAN이나 Routing에 필요한 설정을 구성한다.

#### Flannel CNI Plugin

Pod가 생성될 때 Network 설정을 수행한다.

필요한 작업을 `bridge`, `host-local` 등의 Plugin에 위임하여 Pod Interface와 IP를 구성할 수 있다.

### 특징

- 구조가 비교적 단순함
- VXLAN Overlay 사용 가능
- `host-gw` 방식으로 Routing 기반 구성도 가능
- NetworkPolicy 기능은 Flannel 자체의 주요 기능이 아님

Flannel은 **Pod 간 기본 연결성을 제공하는 것에 집중한 Solution**이라고 볼 수 있다.

---

## 11. Calico

Calico는 **L3 Routing과 NetworkPolicy**에 강점을 가진 Network Solution이다.

### 주요 Component

#### Calico CNI Plugin

Pod의 Network Interface를 구성한다.

#### Calico IPAM

Calico의 IPPool을 이용해 Pod IP를 관리한다.

#### Felix

각 Node에서 실행되며 Routing과 NetworkPolicy 관련 설정을 Linux Kernel에 반영한다.

#### BGP Component

Node 간 Pod Network 경로를 BGP를 통해 공유할 수 있다.

---

### Calico Network

대표적인 구성에서는 각 Node가 자신의 Pod Network를 BGP를 통해 다른 Node에게 알려준다.

```text
Node A
Pod Network
10.244.1.0/24
      │
      │ BGP
      ▼
Node B

"10.244.1.0/24는
 Node A로 보내면 된다."
```

이를 이용해 Packet을 다른 Node까지 Routing할 수 있다.

Calico는 BGP만 사용하는 Solution은 아니다.

환경에 따라 다음과 같은 방식을 사용할 수 있다.

- BGP Routing
- VXLAN
- IP-in-IP
- eBPF Data Plane

따라서 Calico의 특징은 특정 Network 기술 하나보다는 **다양한 Routing 방식과 강력한 NetworkPolicy 기능을 제공한다는 점**에 있다.

---

## 12. Cilium

Cilium은 **eBPF를 기반으로 Network, Security, Service 처리, Observability 기능을 제공하는 Network Solution**이다.

### eBPF란?

**eBPF(extended Berkeley Packet Filter)**는 Linux Kernel 내부에서 작은 Program을 실행할 수 있도록 하는 기술이다.

Network Packet이 Kernel을 지나가는 특정 지점에 eBPF Program을 연결하면 Packet을 검사하거나 전달 방식을 결정할 수 있다.

```text
Packet
  │
  ▼
Linux Kernel
  │
  ├─ eBPF Program
  │     ├─ Routing
  │     ├─ Policy
  │     └─ Load Balancing
  │
  ▼
Destination
```

기존 Linux Network에서 여러 기능을 담당하던 iptables 등의 역할을 eBPF로 처리할 수 있다.

---

### 주요 Component

#### cilium-cni

Pod Network Interface를 Cilium Network에 연결한다.

#### cilium-agent

각 Node에서 실행되면서 eBPF Program과 Map을 관리하고 Network Policy와 Service 등의 상태를 Kernel에 반영한다.

#### cilium-operator

Cluster 전체에서 필요한 IPAM이나 Cilium Resource 관리 작업을 수행한다.

#### Hubble

Cilium Network에서 발생하는 Traffic Flow를 관측하는 기능이다.

---

### 특징

Cilium은 eBPF를 이용해 다음 기능을 제공할 수 있다.

- Pod Network
- NetworkPolicy
- Service Load Balancing
- kube-proxy 대체
- L7 Policy
- Network Observability

Node 간 통신 방식도 하나로 고정되지 않고 VXLAN/Geneve와 같은 Tunnel 방식이나 Native Routing 방식을 선택할 수 있다.

---

## 13. Flannel / Calico / Cilium 비교

| 항목 | Flannel | Calico | Cilium |
| --- | --- | --- | --- |
| 핵심 방향 | Pod Network 연결 | Routing + NetworkPolicy | eBPF 기반 Network 통합 |
| Node 간 전달 | VXLAN, host-gw | BGP, VXLAN, IP-in-IP 등 | Tunnel, Native Routing 등 |
| IPAM | Node Pod CIDR + host-local 계열 | Calico IPAM | 다양한 Cilium IPAM Mode |
| NetworkPolicy | 자체 주요 기능 아님 | 지원 | 지원 |
| kube-proxy 대체 | X | eBPF Mode에서 가능 | 가능 |
| 주요 기술 | VXLAN | L3 Routing, BGP | eBPF |
| 구조 | 비교적 단순 | 다양한 Network 구성 | 기능이 많고 통합 범위가 넓음 |

각 Solution을 하나의 기술과 동일하게 볼 필요는 없다.

예를 들어:

```text
Flannel = 반드시 VXLAN
Calico = 반드시 BGP
Cilium = 반드시 Overlay
```

와 같이 고정되는 것이 아니라 설정과 환경에 따라 Network 구성 방식을 선택할 수 있다.

각 Solution의 **대표적인 설계 방향**을 이해하는 것이 중요하다.

---

## 14. CNI와 kube-proxy

CNI와 kube-proxy는 담당하는 Network 영역이 다르다.

### CNI

Pod 자체를 Network에 연결한다.

```text
Pod
 ↓
Interface
 ↓
Pod IP
 ↓
Pod Network
```

### kube-proxy

Service로 들어온 Traffic을 Backend Pod로 전달할 수 있도록 Node의 Network Rule을 구성한다.

```text
Service ClusterIP
       ↓
Service Rule
       ↓
Backend Pod
```

즉 기본적인 역할을 정리하면:

| | CNI | kube-proxy |
| --- | --- | --- |
| 주요 대상 | Pod Network | Service Network |
| 역할 | Pod Interface, IP, Routing 구성 | Service → Pod 전달 Rule 구성 |
| 주요 시점 | Pod 생성/삭제 | Service/Endpoint 변경 |

Cilium이나 Calico의 eBPF Mode처럼 Network Solution이 Service 처리 기능까지 제공하면 kube-proxy의 역할을 대신할 수도 있다.

---

## 15. 전체 흐름

### Pod가 생성될 때

```text
kubelet
   ↓
CRI
   ↓
Container Runtime
   ↓
CNI Plugin
   ↓
IPAM
   ↓
Pod Interface / IP / Route 구성
   ↓
Pod Network 연결
```

CNI Plugin은 **Pod 하나를 Network에 연결하는 작업**을 담당한다.

---

### Cluster Network가 유지될 때

```text
Kubernetes API
      ↓
CNI Solution의
Agent / Controller
      ↓
Node Network 상태 관리
      │
      ├─ Routing
      ├─ Tunnel
      ├─ NetworkPolicy
      └─ Service 처리
```

Flannel, Calico, Cilium 같은 Primary CNI Solution은 CNI Plugin 외에도 Agent와 Controller를 이용하여 Cluster 전체 Network를 지속적으로 관리한다.

---

### Node 간 Packet 전달

대표적으로 두 가지 관점으로 볼 수 있다.

#### Overlay 방식

```text
Pod A
 ↓
Node A
 ↓
VXLAN으로 Packet Encapsulation
 ↓
Node B
 ↓
Packet Decapsulation
 ↓
Pod B
```

기존 Network는 Node IP만 알면 된다.

#### Routing 방식

```text
Pod A
 ↓
Node A Routing Table
 ↓
Pod B Network → Node B
 ↓
Node B
 ↓
Pod B
```

Node 또는 Network 장비가 Pod Network에 대한 Routing 정보를 알고 있어야 한다.

BGP는 이러한 Routing 정보를 서로 공유하는 데 사용할 수 있는 Protocol이다.

---

## 16. 이번 주 핵심 정리

CNI는 **Pod를 Network에 연결하기 위한 표준 호출 규약**이다.

CNI Plugin은 Pod가 생성될 때 Network Interface, IP, Route 등의 설정을 수행한다.

IPAM은 Pod에 할당할 IP의 사용 상태를 관리한다.

Flannel, Calico, Cilium은 CNI Plugin뿐 아니라 Agent, Controller, IPAM, Routing 등의 기능을 포함하여 Kubernetes Cluster Network를 구현하는 **Primary CNI Solution**이다.

각 Solution은 Node 간 Pod 통신을 서로 다른 방식으로 구성할 수 있다.

- **VXLAN**: 기존 Pod Packet을 Node 간 Packet 안에 넣어서 전달하는 Tunnel 기술
- **BGP**: 어떤 IP Network가 어디에 있는지 서로 알려주는 Routing Protocol
- **eBPF**: Linux Kernel 내부에서 Network Packet 처리 Logic을 실행할 수 있는 기술

대표적인 방향을 정리하면 다음과 같다.

```text
Flannel
→ 단순한 Pod Network 연결
→ 대표적으로 VXLAN Overlay

Calico
→ Routing + NetworkPolicy
→ BGP, VXLAN 등 다양한 방식

Cilium
→ eBPF 기반 Network / Policy / Service / Observability 통합
```

Week 2가 **“Pod가 Network를 어떻게 할당받는가”**를 이해하는 주차였다면, Week 3는 **“그 Pod Network를 Cluster 전체에서 어떤 구조로 구현하고 유지하는가”**를 이해하는 주차라고 정리할 수 있다.

---

## 참고 자료

- CNI Specification  
  https://www.cni.dev/docs/spec/

- CNI GitHub  
  https://github.com/containernetworking/cni

- Flannel  
  https://github.com/flannel-io/flannel

- Calico Documentation  
  https://docs.tigera.io/calico/latest/about/

- Cilium Documentation  
  https://docs.cilium.io/