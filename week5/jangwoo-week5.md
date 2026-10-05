# Week 5. Cilium & eBPF

## 1. 학습 목표

Cilium이 Linux Kernel의 eBPF를 이용해 Pod Network, Service, NetworkPolicy, Observability를 어떻게 처리하는지 이해한다.

- eBPF Program이 Network 경로의 어느 Hook에서 실행되는가?
- `cilium-agent`와 Cilium Operator는 어떤 상태를 관리하는가?
- Cilium은 어떻게 kube-proxy를 대체할 수 있는가?
- Identity 기반 Policy와 Hubble Flow는 Data Path와 어떻게 연결되는가?

> 아래 흐름은 개념을 단순화한 것이다. 실제 Hook과 Interface는 Routing Mode, kube-proxy Replacement, L7 Policy 등의 설정에 따라 달라질 수 있다.

---

## 2. 이전 주차와 연결

Week 3에서는 Cilium의 구성요소를 개괄했고, Week 4에서는 Flannel VXLAN과 Calico Routing을 비교했다.

```text
Flannel
  → Bridge, Route, VXLAN을 이용해 Pod Network 구성

Calico의 대표 구성
  → Host Routing, BGP, Kernel Policy 규칙 이용

Cilium
  → Linux Network 기능과 eBPF Program을 결합해
    Routing, Policy, Service, 관찰 기능 구성
```

Cilium도 veth, Linux Routing, VXLAN 같은 기존 Network 기술을 사용할 수 있다. 핵심 차이는 **Packet 경로의 여러 지점에 eBPF Program을 연결하고, eBPF Map에 저장한 상태를 이용해 동작을 결정한다는 것**이다.

---

## 3. eBPF 기본 개념

eBPF는 **extended Berkeley Packet Filter**에서 시작한 이름이다. 현재의 eBPF는 단순한 Packet Filter를 넘어 Networking, Security, Tracing 등에 사용할 수 있는 Linux Kernel의 범용 실행 기술로 발전했다.

eBPF는 검증된 Program을 Linux Kernel의 특정 **Hook**에서 실행할 수 있게 한다. Hook은 Packet 수신, Interface 통과, 애플리케이션의 Socket 연결처럼 **특정 Kernel Event가 발생했을 때 Program을 실행하도록 연결하는 지점**이다. 문 앞에 설치한 검사대가 사람이 들어오는 순간 동작하는 것과 비슷하다.

Network Packet을 무조건 User Space Process로 전달하지 않고 Kernel 안에서 검사, 차단, Redirect, 주소 변환 등을 수행할 수 있다.

```text
User Space
  cilium-agent
      │ Program Load / Map Update
      ▼
Linux Kernel
  eBPF Program + eBPF Map
      │
      ▼
  실제 Packet 처리
```

예를 들어 Pod가 Packet을 보내면 다음과 같이 처리할 수 있다.

```text
Pod에서 Packet 전송
  → Host 측 veth의 TC Hook 도착
  → 연결된 eBPF Program 실행
      ├─ Policy Map 조회
      ├─ 허용: 다음 Endpoint나 Route로 전달
      └─ 차단: Packet Drop과 Verdict 기록
```

### eBPF Program과 Map


| 구분           | 역할                                              |
| ------------ | ----------------------------------------------- |
| eBPF Program | Hook에서 실행되어 Packet의 처리 방법 결정                    |
| eBPF Map     | Program과 User Space가 공유하는 Kernel의 Key-Value 저장소 |
| Verifier     | Program이 Kernel에서 안전하게 실행될 수 있는지 Load 전에 검사     |


Cilium은 Map에 Endpoint, Security Identity, Policy, Service와 Backend, Connection Tracking 같은 상태를 저장한다. 

Agent가 Map을 갱신하면 eBPF Program은 Packet 처리 중 이 값을 조회한다. [Cilium BPF Architecture](https://docs.cilium.io/en/stable/reference-guides/bpf/architecture/)

### 주요 Network Hook


| Hook            | 실행 위치                                      | Cilium에서 볼 수 있는 활용 예시                   |
| --------------- | ------------------------------------------ | --------------------------------------- |
| XDP             | Network Driver가 Packet을 받은 매우 이른 시점        | 빠른 Drop, Service 가속 기능                  |
| TC              | Network Interface의 Ingress/Egress 경로       | Pod Policy, Forwarding, Service 처리      |
| Socket / cgroup | 애플리케이션의 `connect()`, `sendmsg()` 등에 가까운 시점 | Socket Load Balancing, Socket 단위 Policy |


```text
NIC 수신
  → XDP Hook
  → Linux Network Stack
  → TC Hook
  → Socket
  → Application
```

모든 Packet이 위 Hook을 전부 같은 방식으로 지나는 것은 아니다. 기능과 Traffic 방향에 맞는 Hook을 선택한다. eBPF는 Kernel을 우회하는 기술이 아니라 **Kernel 내부의 처리 지점을 Programmable하게 만드는 기술**이다. [Cilium eBPF Introduction](https://docs.cilium.io/en/stable/network/ebpf/intro/)  

ClusterIP로 요청하는 예를 Hook과 연결하면 다음과 같다.

```text
Application이 ClusterIP로 connect()
  → Socket Hook: Service Map에서 Backend 선택
  → Packet 생성
  → Pod veth의 TC Hook: Identity와 Egress Policy 확인
  → Node 간 Network로 전달
  → 목적지 veth의 TC Hook: Ingress Policy 확인 후 Pod에 전달

외부에서 Node로 Packet이 들어오는 경우
  → 설정에 따라 NIC의 XDP Hook에서 조기 처리 또는 Drop 가능
```

---

## 4. Cilium Architecture

주요 구성요소는 다음과 같다.


| 구성요소            | 범위          | 역할                                                        |
| --------------- | ----------- | --------------------------------------------------------- |
| `cilium-cni`    | Pod 생성·삭제 시 | Local Agent에 Pod Network 구성 요청                            |
| `cilium-agent`  | 각 Node      | Endpoint, Identity, Policy, Service 상태와 eBPF Data Path 관리 |
| Cilium Operator | Cluster     | IPAM 등 Cluster에서 한 번 조정할 작업 처리                            |
| Hubble Server   | 각 Node      | Local eBPF Data Path의 Flow Event 제공                       |
| Hubble Relay    | Cluster     | 여러 Node의 Hubble Server를 연결해 Flow 집계                       |


```text
Kubernetes API
  ├─ Pod / Node
  ├─ Service / EndpointSlice
  ├─ NetworkPolicy
  └─ CiliumNetworkPolicy
          │
          ├───────────────────┐
          ▼                   ▼
 Cilium Operator       Node별 cilium-agent
 Cluster 단위 작업      ├─ Endpoint / Identity 관리
 IPAM 조정 등           ├─ Policy 계산
                       ├─ Service / Backend Map 갱신
                       └─ eBPF Program Load 및 갱신
                                  │
                                  ▼
                         Linux Kernel Data Path
                                  │
                                  ▼
                         Hubble Flow Event
```

Operator와 Agent는 Packet을 직접 중계하지 않는다. 이들은 Kubernetes의 원하는 상태를 Kernel의 eBPF Program과 Map에 반영하는 Control Plane 역할을 한다. 

실제 Packet은 미리 구성된 Kernel Data Path를 지난다. [Cilium Component Overview](https://docs.cilium.io/en/stable/overview/component-overview/)

---

## 5. Pod가 생성될 때 Cilium이 하는 일

```text
1. Scheduler가 Pod를 Node에 배치
2. Runtime이 Pod Network Namespace 준비
3. Runtime이 cilium-cni 실행
4. cilium-cni가 같은 Node의 cilium-agent에 요청
5. Pod eth0과 Host 측 veth 연결
6. IPAM Mode에 따라 Pod IP 할당
7. Agent가 Cilium Endpoint와 Security Identity 준비
8. 해당 Endpoint의 eBPF Program과 Policy Map 구성
```

`cilium-cni`는 Pod의 Network 생명주기에 맞춰 호출되는 Plugin이고, `cilium-agent`는 계속 실행되면서 Kubernetes Event와 Data Path 상태를 관리한다.

```text
Pod Network Namespace                 Host Network Namespace

Pod eth0
   │
   └──────────── veth pair ─────────── lxc* (Host 측 Interface)
                                           │
                                      eBPF Program
                                           │
                                 Routing 또는 Overlay
```

`eth0`은 일반적으로 Pod Network Namespace 안에서 애플리케이션이 사용하는 기본 Network Interface다. `lxc*`는 해당 veth pair의 Host 측 Interface에 Cilium이 흔히 사용하는 이름 형태다. 두 Interface는 veth pair로 연결되어 한쪽으로 들어간 Packet이 다른 쪽으로 나온다. 실제 이름은 환경과 Version에 따라 달라질 수 있다.

**User Space Proxy**는 Kernel 밖의 일반 Process로 실행되면서 전달받은 Traffic을 검사하고 다시 목적지로 보내는 프로그램이다. Cilium에서는 HTTP Method나 Path 같은 L7 Policy가 필요할 때 Cilium이 관리하는 Envoy가 이 역할을 할 수 있다.

```text
일반 L3/L4 Traffic
Pod → veth의 eBPF Program → Routing → 목적지

L7 검사가 필요한 Traffic
Pod → eBPF Program → Envoy로 Redirect
    → HTTP Policy 검사 → 허용 시 목적지로 전달
```

Pod마다 Proxy를 하나씩 두어 모든 Packet을 통과시키는 구조는 아니다. 일반적인 L3/L4 Policy와 Forwarding은 veth와 Kernel Hook의 eBPF Program에서 처리하고, Proxy가 필요한 Traffic만 User Space로 Redirect할 수 있다.

---

## 6. Cilium Pod-to-Pod Data Path

다음과 같은 환경을 가정한다.

```text
Node A: 192.168.10.11     Pod A: 10.0.1.10
Node B: 192.168.10.12     Pod B: 10.0.2.20
```

### 다른 Node의 Pod로 요청

```text
Pod A
  → Host 측 veth의 eBPF Program
      ├─ Source Identity 확인
      ├─ Egress Policy 확인
      └─ 목적지 Endpoint와 전달 경로 조회
  → Native Routing 또는 VXLAN/Geneve Overlay
  → Node B의 eBPF Data Path
      ├─ Source Identity 확인
      ├─ Ingress Policy 확인
      └─ Local Endpoint 조회
  → Pod B
```

- **Native Routing:** 별도 Tunnel 없이 Linux Route와 Underlay를 이용한다. Underlay가 Pod CIDR을 전달할 수 있어야 한다.
- **Overlay:** VXLAN 또는 Geneve로 원래 Pod Packet을 Node IP Packet 안에 넣어 전달한다. Underlay는 Node IP만 알면 된다.

Cilium은 eBPF를 사용하지만 모든 환경에서 동일한 Routing Mode를 강제하지 않는다. Packet의 Node 간 전달 방식과 eBPF 기반 Policy·Forwarding 처리는 구분해서 봐야 한다. [Cilium Introduction](https://docs.cilium.io/en/stable/overview/intro/), [Life of a Packet](https://docs.cilium.io/en/stable/network/ebpf/lifeofapacket/)

---

## 7. kube-proxy Replacement

일반적인 kube-proxy는 Service와 EndpointSlice를 감시하고, ClusterIP를 Backend Pod로 연결할 Kernel 규칙을 각 Node에 구성한다. Cilium은 이 역할을 `cilium-agent`와 eBPF Data Path로 대체할 수 있다.

### Service 상태 준비

```text
Service: 10.96.0.10:80
EndpointSlice:
  ├─ 10.0.1.11:8080
  └─ 10.0.2.20:8080
          │
          ▼
cilium-agent가 Kubernetes 변경 확인
          │
          ▼
eBPF Service / Backend Map 갱신
```

### Client Pod → Service → Backend

```text
Application이 10.96.0.10:80으로 연결 요청
  → Socket Hook에서 Service Map 조회
  → Backend 하나 선택
  → 연결 대상을 10.0.2.20:8080으로 변경
  → Packet 생성
  → Policy 확인과 Pod Network Routing
  → Backend Pod 도착
```

Socket Load Balancing은 TCP `connect()` 또는 UDP `sendmsg()` 등에 가까운 시점에 Backend를 선택한다. 즉 Packet이 완성되기 전에 Service 주소를 Backend 주소로 바꿀 수 있다. Socket Hook을 사용할 수 없는 경로에서는 TC 기반의 Packet Load Balancing을 사용할 수 있다.

```text
kube-proxy의 대표 방식
Service/EndpointSlice → iptables·nftables·IPVS 규칙 → Packet DNAT

Cilium kube-proxy Replacement
Service/EndpointSlice → cilium-agent → eBPF Map → Socket/TC eBPF 처리
```

둘 다 Service를 실제 Backend로 연결하지만, 규칙을 저장하고 실행하는 Data Plane이 다르다. Cilium을 설치했다고 kube-proxy가 자동으로 사라지는 것은 아니며 kube-proxy Replacement 설정을 활성화해야 한다. [Cilium Kubernetes Without kube-proxy](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)

---

## 8. Identity 기반 NetworkPolicy

Pod IP는 재생성이나 Scale Out에 따라 바뀔 수 있다. Cilium은 Kubernetes Label에서 **Security Identity**를 만들고 Policy의 주요 기준으로 사용한다.

```text
Pod A labels: app=frontend
  → Security Identity: 1001

Pod B labels: app=backend
  → Security Identity: 2001

Policy
  Identity 1001 → Identity 2001의 TCP 8080 허용
```

같은 Security 관련 Label 집합을 가진 여러 Pod는 같은 Identity를 공유할 수 있다. 새 `frontend` Pod의 IP가 달라져도 Identity가 같다면 Policy 의미는 유지된다.

### Policy가 Data Path에 반영되는 과정

```text
Kubernetes NetworkPolicy 또는 CiliumNetworkPolicy 생성
  → 각 cilium-agent가 Policy와 Endpoint Label 확인
  → 허용할 Identity, Protocol, Port 계산
  → Endpoint의 eBPF Policy Map 갱신
  → Packet이 Hook을 지날 때 Source Identity와 Policy 조회
  → FORWARDED 또는 DROPPED 결정
```

Cilium은 Kubernetes NetworkPolicy를 지원하고, CiliumNetworkPolicy로 DNS나 HTTP 같은 확장 기능도 제공한다. L7 HTTP Policy가 필요하면 Envoy Proxy로 Traffic을 Redirect해 Method나 Path를 검사할 수 있다. 모든 Traffic이 항상 Envoy를 지나는 것은 아니다. [Cilium Identity-Based Security](https://docs.cilium.io/en/stable/security/network/identity/), [Cilium Network Policy](https://docs.cilium.io/en/stable/security/policy/)

---

## 9. Hubble: Data Path와 Observability의 연결

Hubble은 Cilium eBPF Data Path에서 발생한 Flow Event를 이용해 Network와 Policy 동작을 보여준다.

```text
Packet이 eBPF Data Path를 지남
  → Source / Destination Identity 확인
  → Policy Verdict와 전달 지점 기록
  → Node의 Hubble Server가 Flow 제공
  → Hubble Relay가 여러 Node의 Flow 집계
  → Hubble CLI 또는 UI로 조회
```


| 범위      | 구성요소            | 확인할 수 있는 예시                    |
| ------- | --------------- | ------------------------------ |
| Node    | Hubble Server   | 해당 Node가 관찰한 Flow              |
| Cluster | Hubble Relay    | 여러 Node의 Flow를 한 API로 조회       |
| 사용자     | Hubble CLI / UI | Pod, Namespace, Verdict 등으로 검색 |


예를 들어 다음과 같은 질문을 확인할 수 있다.

- 어느 Pod가 어떤 Service 또는 Pod에 연결했는가?
- NetworkPolicy 때문에 `DROPPED`된 Flow가 있는가?
- 요청이 `FORWARDED`되었는가?
- L7 가시성이 설정된 경우 어떤 HTTP Method와 Path가 사용됐는가?

Hubble은 모든 Packet Payload를 저장하는 일반적인 Packet Capture와는 다르다. Cilium이 알고 있는 Endpoint·Identity·Policy 정보를 Flow Event와 연결해 보여주는 Observability 도구다. [Hubble Overview](https://docs.cilium.io/en/stable/observability/hubble/index.html), [Hubble CLI](https://docs.cilium.io/en/stable/observability/hubble/hubble-cli/)

---

## 10. 하나의 요청으로 전체 흐름 연결

`frontend` Pod가 ClusterIP를 통해 다른 Node의 `backend` Pod로 요청한다고 가정한다.

```text
1. frontend Application이 ClusterIP:80으로 connect()
2. Socket eBPF가 Service Map에서 Backend Pod:8080 선택
3. frontend의 Security Identity와 Egress Policy 확인
4. Packet을 Native Routing 또는 Overlay로 목적지 Node에 전달
5. 목적지 Node에서 backend의 Ingress Policy 확인
6. 허용되면 Backend Pod에 전달, 아니면 Drop
7. eBPF Data Path가 Verdict를 Flow Event로 생성
8. Hubble에서 Source, Destination, Port, Verdict 확인
```

각 기능은 따로 떨어져 있지 않다.


| 기능                     | 사용하는 상태                   | 실행되는 곳                             |
| ---------------------- | ------------------------- | ---------------------------------- |
| Service Load Balancing | Service / Backend Map     | Socket 또는 TC eBPF Hook             |
| NetworkPolicy          | Identity / Policy Map     | Endpoint의 eBPF Hook, 필요 시 L7 Proxy |
| Pod Forwarding         | Endpoint / Route 정보       | Linux Routing과 eBPF Data Path      |
| Observability          | Data Path가 생성한 Flow Event | Hubble Server / Relay              |


---

## 11. 기존 CNI 방식과 비교


| 구분        | 전통적인 대표 구성                               | Cilium                           |
| --------- | ---------------------------------------- | -------------------------------- |
| Pod 연결    | veth, Bridge 또는 Host Routing             | veth와 Routing에 eBPF Data Path 결합 |
| Service   | kube-proxy가 iptables·nftables·IPVS 규칙 구성 | 선택적으로 eBPF가 kube-proxy 역할 대체     |
| Policy 기준 | 주로 IP, CIDR, Port와 Label Selector의 변환 결과 | Label에서 만든 Security Identity 중심  |
| Packet 처리 | Linux의 고정된 Network 기능과 규칙                | 여러 Kernel Hook에서 eBPF Program 실행 |
| 관찰        | tcpdump, conntrack, 별도 Monitoring 도구     | Hubble이 eBPF Flow와 Cilium 상태 연결  |


이 비교는 이해를 위한 대표 형태다. 다른 CNI도 eBPF를 사용할 수 있고, Cilium도 Linux Routing·Netfilter·Tunnel 같은 기존 기능을 함께 사용한다. 따라서 `기존 CNI는 eBPF를 쓰지 않는다`거나 `Cilium은 iptables와 Routing을 전혀 사용하지 않는다`고 일반화하면 안 된다.

---

## 12. 학습하면서 생긴 질문

### Q1. Identity 기반 Policy라면 Pod가 새로 생성될 때마다 모든 Node의 Policy를 다시 만들어야 할까?

새 Pod가 기존 Pod와 같은 Security 관련 Label 집합을 가지면 같은 Identity를 공유할 수 있다. Policy는 개별 Pod IP 목록보다 Identity 사이의 관계로 표현되므로, 새 Replica가 추가될 때 모든 목적지 Node의 Policy 의미를 Pod IP별로 다시 작성할 필요를 줄일 수 있다. 다만 Agent는 새 Endpoint의 IP와 Identity 관계를 Data Path에 반영해야 한다.  

```text
기존 frontend Replica
  Pod A: 10.0.1.10, labels app=frontend → Identity 1001
  Pod B: 10.0.2.11, labels app=frontend → Identity 1001

Policy
  Identity 1001 → Identity 2001(backend)의 TCP 8080 허용

새 frontend Replica 생성
  Pod C: 10.0.3.15, labels app=frontend → Identity 1001
       │
       ├─ 새 IP와 Identity 관계는 Data Path에 반영
       └─ backend Policy의 의미는 그대로 유지
```

### Q2. Socket Load Balancing을 사용하면 Pod 내부의 `tcpdump`에서 ClusterIP가 보이지 않을 수도 있을까?

그럴 수 있다. Socket Hook에서 애플리케이션의 연결 대상을 Backend로 바꾸면 Network Packet이 만들어질 때는 이미 Destination이 Backend IP일 수 있다. 따라서 Capture 위치에 따라 ClusterIP가 아니라 Backend IP가 보인다. TC 기반 Load Balancing처럼 Packet 생성 이후 변환하는 경로에서는 관찰 결과가 달라질 수 있다.

---

## 13. 용어 설명집


| 용어                     | 설명                                                      |
| ---------------------- | ------------------------------------------------------- |
| eBPF                   | extended Berkeley Packet Filter에서 시작한 이름으로, Kernel Hook에서 검증된 Program을 실행하는 기술 |
| Hook                   | 특정 Kernel Event가 발생할 때 eBPF Program이 실행되는 지점            |
| XDP                    | Network Driver의 이른 수신 단계에서 실행되는 eBPF Hook               |
| TC                     | Network Interface의 Traffic Control 경로에서 사용하는 Hook       |
| eBPF Map               | eBPF Program과 User Space가 상태를 공유하는 Kernel Key-Value 저장소 |
| veth pair              | 두 Network Namespace를 가상 Cable처럼 연결하는 Interface 한 쌍       |
| User Space Proxy       | Kernel 밖의 Process로 실행되며 전달받은 Traffic을 검사하고 중계하는 Proxy |
| Envoy                  | Cilium이 L7 HTTP Policy 등의 처리에 사용할 수 있는 User Space Proxy   |
| Endpoint               | Cilium이 Network와 Policy를 관리하는 Pod 등의 통신 주체              |
| Security Identity      | Security 관련 Label 집합에서 만들어진 숫자 식별자                      |
| kube-proxy Replacement | Cilium eBPF가 Kubernetes Service 전달 기능을 대신하는 구성          |
| Policy Verdict         | Policy 검사 결과인 허용 또는 차단 결정                               |
| Hubble                 | Cilium Data Path의 Flow를 수집하고 조회하는 Observability 기능      |
| Hubble Relay           | 여러 Node의 Hubble Server를 연결해 Cluster 단위 조회를 제공하는 구성요소    |


---

## 14. 참고 자료

- [Cilium Component Overview](https://docs.cilium.io/en/stable/overview/component-overview/)
- [Cilium eBPF Introduction](https://docs.cilium.io/en/stable/network/ebpf/intro/)
- [Cilium BPF Architecture](https://docs.cilium.io/en/stable/reference-guides/bpf/architecture/)
- [Cilium Life of a Packet](https://docs.cilium.io/en/stable/network/ebpf/lifeofapacket/)
- [Cilium Kubernetes Without kube-proxy](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)
- [Cilium Identity-Based Security](https://docs.cilium.io/en/stable/security/network/identity/)
- [Cilium Network Policy](https://docs.cilium.io/en/stable/security/policy/)
- [Hubble Overview](https://docs.cilium.io/en/stable/observability/hubble/index.html)
- [Hubble CLI](https://docs.cilium.io/en/stable/observability/hubble/hubble-cli/)
