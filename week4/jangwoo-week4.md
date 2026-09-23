# Week 4. Primary CNI: Flannel과 Calico의 실제 통신 흐름

## 1. 학습 목표

Week 3에서는 Flannel과 Calico의 구성요소를 살펴봤다. 이번에는 **같은 Pod-to-Pod 통신이 두 CNI에서 어떤 경로로 전달되는지** 비교한다.

- Flannel의 기본 VXLAN 구성과 Calico의 Routing/BGP 구성 이해
- Overlay와 Underlay에서 Packet의 IP가 어떻게 보이는지 비교
- NetworkPolicy와 IPAM이 어느 단계에서 처리되는지 구분

> 아래 IP와 Route는 흐름을 설명하기 위한 예시다. 실제 Cluster의 Interface 이름, CIDR, Data Plane은 설치 설정에 따라 다를 수 있다.

---

## 2. 먼저 구분할 것: 설정 경로와 Packet 경로

CNI Plugin과 Node Agent는 **통신에 필요한 상태를 준비**한다. 실제 Packet은 매번 이 Process들을 통과하지 않고 Linux Kernel의 Interface, Route, Tunnel, Policy 규칙을 따라 이동한다.

```text
설정 경로: Kubernetes 상태 → CNI Plugin / Node Agent → Kernel Network 구성
Packet 경로: Pod → veth → Node의 Kernel → 다른 Node → veth → Pod
```

두 가지 Network 방식도 구분한다.

| 방식 | Node 사이에서 보이는 Packet | Underlay에 필요한 것 |
|---|---|---|
| Overlay | 원래 Pod Packet을 Node IP Packet으로 감싼 형태 | Node IP 사이의 통신 |
| Non-overlay | Pod IP를 그대로 가진 Packet | 목적지 Pod CIDR로 가는 Route |

**Underlay**는 Node를 실제로 연결하는 Network다. **Overlay**는 그 위에 Tunnel을 만들어 Pod Network를 구현한다. VXLAN은 대표적인 Overlay 방식이며, 바깥쪽 Node IP로 전달한 뒤 목적지 Node에서 원래 Pod Packet을 꺼낸다.

---

## 3. 비교에 사용할 예시

```text
Node A IP: 192.168.10.11       Node B IP: 192.168.10.12
Pod CIDR: 10.244.1.0/24        Pod CIDR: 10.244.2.0/24
Pod A:    10.244.1.5           Pod B:    10.244.2.8
```

Pod A가 Pod B의 IP `10.244.2.8`로 직접 요청한다고 가정한다. Service IP를 거치지 않으므로 이 예시의 핵심은 kube-proxy가 아니라 **Pod Network의 전달 경로**다.

Kubernetes Network Model에서는 서로 다른 Node의 Pod도 Pod IP로 통신할 수 있어야 한다. 두 CNI가 해결해야 할 공통 질문은 다음과 같다.

1. Pod A와 Pod B에 중복되지 않는 IP를 어떻게 할당하는가?
2. Node A는 `10.244.2.8`이 Node B에 있다는 것을 어떻게 아는가?
3. Node 사이를 지나는 Packet은 어떤 형태인가?
4. NetworkPolicy가 있다면 어디에서 적용하는가?

---

## 4. Flannel: Node 간 VXLAN 통신

Flannel은 기본 Pod 연결을 제공하는 데 집중한다. 일반적인 Kubernetes 구성에서 `flanneld`는 Node가 사용할 Pod Subnet을 확인하고 VXLAN 장치와 경로를 준비한다. Flannel CNI Plugin은 Pod가 생성될 때 해당 Node의 Subnet 정보를 읽어, 기본적으로 `bridge` Plugin과 `host-local` IPAM에 Pod 연결을 위임한다.

### 네트워크가 준비되는 과정

```text
Node별 flanneld
  → Node A: 10.244.1.0/24, Node B: 10.244.2.0/24 확인
  → 다른 Node의 Subnet과 Node IP 정보 확인
  → VXLAN 장치(flannel.1)와 Route 준비
  → /run/flannel/subnet.env에 Subnet, MTU 등 기록

Pod A 생성
  → Runtime이 Flannel CNI Plugin 호출
  → bridge Plugin이 Pod eth0 ↔ veth ↔ cni0 Bridge 연결
  → host-local IPAM이 10.244.1.0/24에서 Pod IP 할당
```

여기서 **Node Subnet 배정**과 **개별 Pod IP 할당**은 다른 단계다. `flanneld`가 Node의 대역을 정하고, `host-local`이 그 대역 안에서 Pod A의 IP를 선택한다. [Flannel 문서](https://github.com/flannel-io/flannel/blob/master/README.md), [Flannel CNI Plugin 문서](https://github.com/flannel-io/cni-plugin)

### Pod A → Pod B Packet 흐름

```text
Pod A 10.244.1.5
  → Pod A의 eth0 / veth
  → Node A의 cni0 Bridge
  → Node A가 10.244.2.0/24 경로 조회
  → flannel.1에서 VXLAN으로 감쌈
       Inner: 10.244.1.5 → 10.244.2.8
       Outer: 192.168.10.11 → 192.168.10.12
  → Underlay Network는 Outer Node IP로 Node B까지 전달
  → Node B의 flannel.1에서 VXLAN Header 제거
  → cni0 Bridge / veth → Pod B 10.244.2.8
```

Node 사이의 VXLAN Packet은 개념적으로 `Outer Ethernet/IP/UDP/VXLAN Header + Inner Ethernet/IP Packet`으로 구성된다. Pod A와 Pod B는 Inner IP로 통신하지만, Underlay에서는 Outer IP인 두 Node의 주소를 사용한다.

Underlay는 Pod CIDR을 몰라도 된다. 다만 VXLAN Header가 추가되므로 MTU를 고려해야 하고, Node 사이에서 VXLAN Traffic이 허용되어야 한다. Flannel은 `host-gw`처럼 Tunnel 없이 Node를 다음 Hop으로 사용하는 Backend도 지원하지만, 여기서는 대표적인 VXLAN 구성을 기준으로 비교한다. [Flannel Backend 문서](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md)

---

## 5. Calico: Routing과 BGP를 이용한 통신

Calico는 Pod를 Host의 Layer 3 Routing에 연결한다. 대표적인 Non-overlay/BGP 구성에서는 Calico CNI가 Pod와 Host 사이에 veth를 만들고 IP를 할당한다. 각 Node의 Felix는 Local Pod Route와 NetworkPolicy 규칙을 Kernel에 반영한다. BIRD는 BGP를 사용해 다른 Node 또는 Router에 Pod 대역의 경로를 알린다.

### 네트워크가 준비되는 과정

```text
Pod A 생성
  → Runtime이 Calico CNI Plugin 호출
  → Calico IPAM이 IP Pool에서 10.244.1.5 할당
  → Pod eth0 ↔ veth ↔ Node A의 Host Routing 연결
  → Felix가 Local Pod Route와 Policy 규칙 구성

Node 간 경로 공유
  → BIRD가 BGP Peer와 Pod CIDR Route 교환
  → Node A가 10.244.2.0/24의 다음 Hop을 알게 됨
```

Flannel 예시의 `cni0` Bridge와 달리, 이 Calico 구성에서는 Host의 Layer 3 Route가 Pod 연결의 중심이다. **BGP는 경로 정보를 전달하는 Control Plane**이다. 실제 Pod Packet이 BIRD Process를 통과하거나 BGP Packet으로 변하는 것은 아니다. [Calico 구성요소 문서](https://docs.tigera.io/calico/latest/reference/architecture/overview), [Calico Data Path](https://docs.tigera.io/calico/latest/reference/architecture/data-path)

### Pod A → Pod B Packet 흐름: Non-overlay/BGP 예시

```text
Pod A 10.244.1.5
  → Pod A의 eth0 / Host 측 veth
  → 정책이 설정돼 있다면 Node A에서 Egress 검사
  → Kernel이 10.244.2.0/24의 Route 조회
  → 다음 Hop(Node B 또는 Underlay Router)으로 전달
       Packet IP: 10.244.1.5 → 10.244.2.8 그대로 유지
  → 정책이 설정돼 있다면 Node B에서 Ingress 검사
  → Local Pod Route / veth → Pod B 10.244.2.8
```

두 Node가 같은 L2 Network에 있는 이 예시에서는 Node A가 `10.244.2.0/24 → 192.168.10.12(Node B)`와 같은 경로를 사용할 수 있다. 이 경로는 **BGP로 배운 목적지 정보**이고, Packet 자체는 원래 Pod IP를 유지한다.

이 방식에는 목적지 Pod CIDR을 전달할 수 있는 Underlay 경로가 필요하다. 같은 L2 Network라면 Node B를 다음 Hop으로 사용할 수 있고, L3 Network라면 중간 Router에도 필요한 Route가 있어야 한다. BGP는 이러한 Route를 교환하는 한 방법이다. [Calico BGP 문서](https://docs.tigera.io/calico/latest/networking/configuring/bgp)

> Calico가 항상 BGP와 Non-overlay를 사용하는 것은 아니다. VXLAN/IP-in-IP Overlay나 Cross-subnet Mode도 지원한다. 특히 VXLAN IP Pool에서는 Felix가 Cluster Route를 구성할 수 있어 내부 통신에 BGP가 필수는 아니다. [Calico Overlay 문서](https://docs.tigera.io/calico/latest/networking/configuring/vxlan-ipip)

---

## 6. 같은 Pod 통신의 차이

위 예시처럼 **서로 다른 Node**에 있는 Pod끼리 통신할 때의 대표 경로를 나란히 놓으면 다음과 같다.

| 단계 | Flannel VXLAN | Calico Non-overlay/BGP |
|---|---|---|
| Pod와 Host 연결 | 보통 veth → `cni0` Bridge | veth → Host의 L3 Routing |
| 목적지 Node 정보 | Flannel의 Subnet/Backend 정보 | BGP 등으로 교환한 Pod CIDR Route |
| Node 간 전달 | VXLAN으로 캡슐화 | Pod IP Packet을 그대로 Routing |
| Underlay가 보는 목적지 | Node B IP | Pod B IP |
| Underlay 요구사항 | Node IP 통신과 VXLAN 허용 | Pod CIDR Routing 가능 |
| 정책 적용 | Flannel 자체 Agent에는 정책 적용 기능 없음 | Felix가 구성한 Data Plane 규칙 사용 |

**같은 Node**의 Pod 통신은 Node 간 VXLAN이나 BGP Route가 필요하지 않다. Flannel의 기본 Bridge 구성이라면 `Pod A → veth → cni0 → veth → Pod B`, Calico의 대표 구성이라면 `Pod A → veth → Host Routing → veth → Pod B`로 이해할 수 있다.

따라서 `Flannel = 항상 Overlay`, `Calico = 항상 Underlay`로 외우면 틀릴 수 있다. 여기서 비교한 것은 **Flannel VXLAN**과 **Calico Non-overlay/BGP**라는 두 가지 구체적인 설정이다.

---

## 7. IPAM과 NetworkPolicy는 어디에서 처리될까?

### IPAM: Pod 생성 시 주소 결정

IPAM은 Packet이 지나갈 때마다 실행되는 기능이 아니다. Pod Network를 만드는 과정에서 IP를 할당하고, Pod가 제거되면 해당 할당을 정리한다.

| 구분 | Flannel의 기본 구성 | Calico의 일반적인 구성 |
|---|---|---|
| Node가 사용할 대역 | `flanneld`가 Subnet 정보 관리 | Calico IP Pool에서 Node별 Block 할당 가능 |
| 개별 Pod IP | 주로 `host-local` IPAM | 주로 Calico IPAM |
| 적용 시점 | CNI `ADD` / `DEL` | CNI `ADD` / `DEL` |

Calico도 다른 IPAM Plugin과 조합할 수 있으므로 위 표는 대표 구성이다. [Calico IPAM 문서](https://docs.tigera.io/calico/latest/networking/ipam/get-started-ip-addresses)

### NetworkPolicy: 통신 허용 여부 결정

Kubernetes `NetworkPolicy`는 Pod의 Ingress/Egress Traffic에 대한 허용 규칙을 선언한다. 이 API를 실제 Packet 처리 규칙으로 바꾸는 Network Plugin이 있어야 효력이 있다. [Kubernetes NetworkPolicy 문서](https://kubernetes.io/docs/concepts/services-networking/network-policies/)

```text
NetworkPolicy Object 생성
  → Policy 구현체가 규칙을 읽음
  → 각 Node의 Kernel Data Plane에 허용/차단 규칙 구성
  → Pod Packet이 지날 때 해당 규칙 검사
```

- **Flannel:** `flanneld` 자체는 NetworkPolicy를 적용하지 않는다. 필요한 경우 별도 Policy Controller나 Calico와의 조합이 필요하다.
- **Calico:** Felix가 NetworkPolicy를 Node의 Data Plane 규칙으로 만든다. 위 예시에서는 Pod A의 Egress와 Pod B의 Ingress 경로에서 정책이 적용될 수 있다.

즉 **IPAM은 주소를 정하는 단계**, **NetworkPolicy는 통신할 때 허용 여부를 판단하는 단계**에 해당한다. Flannel의 Policy 지원 범위는 설치한 별도 구성요소에 따라 달라진다. [Flannel 공식 문서](https://github.com/flannel-io/flannel/blob/master/README.md), [Calico 구성요소 문서](https://docs.tigera.io/calico/latest/reference/architecture/overview)

---

## 8. 학습하면서 생긴 질문

### Q1. Calico에서 BGP Peer가 끊기면 이미 실행 중인 Pod의 Interface도 사라질까?

아니다. Pod의 veth와 IP는 별도로 구성되어 있다. 다만 BGP에 의존해 다른 Node의 Pod CIDR Route를 배웠다면, 경로가 철회되거나 더 이상 갱신되지 않아 **다른 Node의 Pod와 통신할 수 없게 될 수 있다**. 같은 Node의 Pod 통신과 다른 Node의 Pod 통신을 나눠서 봐야 한다.

### Q2. Flannel에 NetworkPolicy Object만 생성하면 Pod Traffic이 차단될까?

아니다. Kubernetes API에 Object가 저장되는 것과 Packet 필터링은 별개다. Flannel Networking만 있고 NetworkPolicy를 구현하는 Controller가 없다면 해당 정책은 효과가 없다. 별도 Policy 구현체를 함께 설치해야 한다.

---

## 9. 용어 설명집

| 용어 | 설명 |
|---|---|
| Primary CNI | Pod의 기본 Network 연결을 제공하는 CNI 기반 Solution |
| Subnet Lease | Flannel이 Node에 배정한 Pod Subnet과 Node 도달 정보 |
| IPAM | Pod에 사용할 IP의 할당과 반환을 관리하는 기능 |
| Underlay | Node 사이를 실제로 연결하는 기반 Network |
| Overlay | Underlay 위에서 Packet을 캡슐화해 구성한 논리 Network |
| VXLAN | Ethernet Frame을 UDP Packet 안에 넣어 전달하는 Overlay 기술 |
| BGP | Node 또는 Router 사이에서 Route 정보를 교환하는 Protocol |
| Felix | Calico의 Node Agent로, Route와 Policy 규칙을 Kernel에 반영 |
| BIRD | Calico의 BGP Mode에서 Route 정보를 Peer와 교환하는 Daemon |
| NetworkPolicy | Pod의 Ingress/Egress 허용 Traffic을 선언하는 Kubernetes API |
| MTU | Network Interface가 한 번에 전달할 수 있는 Packet 크기의 상한 |

---

## 10. 참고 자료

- [Kubernetes Network Model](https://kubernetes.io/docs/concepts/services-networking/)
- [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Flannel](https://github.com/flannel-io/flannel/blob/master/README.md)
- [Flannel CNI Plugin](https://github.com/flannel-io/cni-plugin)
- [Flannel Backends](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md)
- [Calico Component Architecture](https://docs.tigera.io/calico/latest/reference/architecture/overview)
- [Calico Data Path](https://docs.tigera.io/calico/latest/reference/architecture/data-path)
- [Calico BGP](https://docs.tigera.io/calico/latest/networking/configuring/bgp)
- [Calico Overlay Networking](https://docs.tigera.io/calico/latest/networking/configuring/vxlan-ipip)
- [Calico IPAM](https://docs.tigera.io/calico/latest/networking/ipam/get-started-ip-addresses)
