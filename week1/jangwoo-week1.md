# Week 1. Kubernetes & Linux Networking Overview

## 1. 학습 목표

Kubernetes에서 Pod가 자신만의 IP를 가지고 통신할 수 있는 이유를 Linux 네트워크 기능과 연결해서 이해한다.

- Kubernetes 컴포넌트의 기본 역할
- Network Namespace, veth pair, Linux Bridge
- TCP/IP Stack, Routing, ARP
- Netfilter와 iptables
- 같은 Node에 있는 Pod 사이의 패킷 흐름

> Kubernetes는 네트워크가 만족해야 할 모델을 정의하고, 실제 구현은 CNI 플러그인이 담당한다. 이 문서에서는 이해하기 쉬운 bridge 기반 구성을 예로 사용한다.

---

## 2. Kubernetes 컴포넌트

Kubernetes 클러스터는 **Control Plane**과 **Worker Node**로 구성된다.

### Control Plane

| 컴포넌트 | 역할 |
|---|---|
| `kube-apiserver` | Kubernetes API 제공 |
| `etcd` | 클러스터 상태 저장 |
| `kube-scheduler` | Pod가 실행될 Node 결정 |
| `kube-controller-manager` | 원하는 상태와 현재 상태가 일치하도록 조정 |

### Worker Node

| 컴포넌트 | 역할 |
|---|---|
| `kubelet` | Node에서 Pod가 정상적으로 실행되도록 관리 |
| Container Runtime | 컨테이너 실행과 생명주기 관리 |
| CNI Plugin | Pod의 interface, IP, route 등 네트워크 구성 |
| `kube-proxy` | Service 트래픽을 위한 Node의 네트워크 규칙 관리 |

Pod 네트워크 구성 과정을 단순화하면 다음과 같다.

```text
kubelet
  → Container Runtime
    → Pod Network Namespace 준비
      → CNI Plugin을 통한 interface, IP, route 구성
```

### Kubernetes Network Model

Kubernetes의 기본 네트워크 모델은 다음을 요구한다.

- 각 Pod는 클러스터 안에서 고유한 IP를 가진다.
- 한 Pod의 컨테이너들은 네트워크 환경을 공유하고 `localhost`로 통신한다.
- 서로 다른 Pod는 같은 Node인지와 관계없이 Pod IP로 통신할 수 있어야 한다.
- Pod 간 통신에는 원칙적으로 NAT가 필요하지 않아야 한다.

이 요구사항을 bridge, routing, overlay, eBPF 등 어떤 방식으로 구현할지는 CNI 플러그인에 따라 달라진다.

---

## 3. TCP/IP Stack

애플리케이션 데이터가 네트워크로 전송될 때는 계층별 header가 붙는다. 이를 **캡슐화**라고 한다. 수신 측에서는 반대로 header를 제거하며 데이터를 애플리케이션에 전달한다.

```text
Application Data
      ↓
TCP/UDP: Port
      ↓
IP: IP Address
      ↓
Ethernet: MAC Address
      ↓
Network Interface
```

| 계층 | 역할 |
|---|---|
| Application | HTTP, DNS 등 애플리케이션 통신 |
| Transport | Port를 이용한 프로세스 간 통신 |
| Internet | IP 주소 지정과 Routing |
| Link | MAC 주소를 이용한 같은 네트워크 구간의 Frame 전달 |

---

## 4. Network Namespace

Network Namespace는 하나의 Linux Kernel 안에서 **독립된 네트워크 환경을 만드는 기능**이다.

Namespace마다 다음 네트워크 자원을 따로 가질 수 있다.

- Network Interface
- IP Address
- Routing Table
- ARP / Neighbor Table
- Firewall Rule
- Socket과 Port

일반적인 Pod는 자신만의 Network Namespace를 사용한다. 따라서 Node와 다른 IP, interface, routing table을 가질 수 있다.

같은 Pod 안의 여러 컨테이너는 하나의 Network Namespace를 공유한다. 그래서 같은 IP를 사용하며 `localhost`로 서로 통신할 수 있다.

하지만 Namespace는 네트워크를 **격리**할 뿐 외부와 연결해 주지는 않는다. 격리된 Pod를 Node의 네트워크에 연결할 때 대표적으로 veth pair를 사용한다.

---

## 5. veth Pair

veth는 **Virtual Ethernet Device**의 약자다. 항상 두 개가 한 쌍으로 생성되며, 한쪽으로 들어간 패킷은 반대쪽으로 전달된다.

가상 Ethernet Cable의 양 끝이라고 생각할 수 있다.

```text
Pod Network Namespace              Host Network Namespace

      eth0        ←── veth pair ──→      vethXXXX
  10.244.1.2
```

일반적인 구성에서는 다음과 같이 사용한다.

1. veth pair의 한쪽을 Pod Network Namespace에 넣는다.
2. Pod 쪽 interface에 IP와 route를 설정한다.
3. 반대쪽은 Host Network Namespace에 둔다.
4. Host 쪽 veth를 bridge나 routing 경로에 연결한다.

Pod의 `eth0`는 Node의 물리 `eth0`와 직접 연결된 것이 아니라, Host에 있는 veth의 반대쪽과 연결되어 있다.

---

## 6. Linux Bridge

Linux Bridge는 Kernel에서 동작하는 **Software L2 Switch**다.

여러 interface를 하나의 L2 네트워크로 연결하고 destination MAC 주소에 따라 Ethernet Frame을 전달한다.

| 물리 네트워크 | Linux 네트워크 |
|---|---|
| Ethernet Cable | veth pair |
| L2 Switch | Linux Bridge |
| MAC Address Table | FDB |

여러 Pod의 Host 측 veth를 하나의 bridge에 연결하면 같은 Node의 Pod들이 통신할 수 있다.

```text
Pod A                                      Pod B
 eth0                                       eth0
   │                                          │
 veth                                       veth
   └──────────── Linux Bridge ────────────────┘
```

Bridge는 source MAC과 port를 학습해 FDB(Forwarding Database)에 저장한다. 이후 destination MAC을 보고 알맞은 port로 Frame을 보낸다. 목적지 MAC을 아직 모르면 여러 port로 Frame을 전달한다.

모든 CNI가 Linux Bridge를 사용하는 것은 아니다. 일부 CNI는 Host Routing이나 eBPF를 이용한다.

---

## 7. Routing과 ARP

### Routing

Routing은 destination IP를 보고 패킷을 어느 interface와 next hop으로 보낼지 결정하는 과정이다.

```text
10.244.1.0/24 dev eth0
default via 10.244.1.1 dev eth0
```

- `10.244.1.3`으로 보낼 때는 같은 subnet이므로 `eth0`로 직접 보낸다.
- 외부 IP로 보낼 때는 default gateway인 `10.244.1.1`로 보낸다.

Routing Table은 다음 질문에 답한다.

> 이 destination IP로 가려면 어느 interface와 next hop을 사용해야 하는가?

### ARP

Ethernet Frame을 보내려면 next hop의 MAC 주소가 필요하다. IPv4 환경에서 ARP는 **IP 주소에 해당하는 MAC 주소를 알아내는 프로토콜**이다.

1. 송신자가 ARP Request를 Broadcast한다.
2. 해당 IP를 가진 장치가 자신의 MAC 주소를 응답한다.
3. 송신자는 결과를 Neighbor Table에 저장한다.

같은 subnet이면 최종 목적지의 MAC 주소를 사용한다. 다른 subnet이면 IP packet의 목적지 IP는 그대로 두고, Ethernet Frame에는 gateway의 MAC 주소를 넣는다.

| 구분 | 저장하는 관계 |
|---|---|
| Routing Table | `IP 대역 → Next Hop / Interface` |
| ARP / Neighbor Table | `IP → MAC` |
| Bridge FDB | `MAC → Port` |

---

## 8. Netfilter와 iptables

### Netfilter

Netfilter는 Linux Kernel의 패킷 처리 경로에 규칙을 적용할 수 있도록 hook을 제공하는 framework다. 방화벽, NAT, packet filtering 등에 사용된다.

```text
외부에서 수신
      ↓
 PREROUTING
      ↓
 Routing 결정
   ┌──┴──────────────┐
   ↓                 ↓
 INPUT             FORWARD
   ↓                 ↓
로컬 프로세스      POSTROUTING → 외부로 송신

로컬 프로세스 → OUTPUT → POSTROUTING → 외부로 송신
```

| Chain | 대상 패킷 |
|---|---|
| `PREROUTING` | 들어온 뒤 Routing 결정 전 |
| `INPUT` | Local Process로 들어가는 패킷 |
| `FORWARD` | Host를 통과해 다른 곳으로 전달되는 패킷 |
| `OUTPUT` | Local Process가 생성한 패킷 |
| `POSTROUTING` | 외부 interface로 나가기 직전 |

### iptables

iptables는 사용자가 Netfilter 규칙을 설정하기 위한 도구다.

```text
iptables 명령 → Netfilter Rule 설정 → Kernel이 패킷 처리
```

주요 table은 다음과 같다.

| Table | 역할 |
|---|---|
| `filter` | 패킷 허용 또는 차단 |
| `nat` | Source/Destination 주소 변환 |
| `mangle` | Packet Header나 Mark 변경 |

Routing Table이 **패킷을 어디로 보낼지** 결정한다면, Netfilter와 iptables는 **그 패킷을 허용·차단하거나 변환할지** 결정한다.

---

## 9. 같은 Node의 Pod 간 패킷 흐름

Pod A와 Pod B가 같은 Linux Bridge에 연결되어 있다고 가정한다.

```text
Pod A: 10.244.1.2
Pod B: 10.244.1.3
```

1. Pod A의 애플리케이션이 데이터를 생성한다.
2. TCP/UDP와 IP 계층을 거쳐 packet이 만들어진다.
3. Routing Table을 확인하고 `eth0`로 보낼 것을 결정한다.
4. Pod B의 MAC 주소를 모르면 ARP로 확인한다.
5. Ethernet Frame이 `eth0`와 veth pair를 통해 Host로 이동한다.
6. Linux Bridge가 destination MAC을 보고 Pod B 측 veth로 전달한다.
7. Pod B에서 Frame과 Packet을 해석해 애플리케이션에 전달한다.

```text
Pod A Process
  → TCP/UDP → IP → Ethernet
  → eth0 → veth
  → Linux Bridge
  → veth → eth0
  → Ethernet → IP → TCP/UDP
  → Pod B Process
```

이 흐름은 bridge 기반의 단순한 예다. 실제 패킷 경로는 사용하는 CNI에 따라 달라질 수 있다.

---

## 10. 핵심 정리

| 요소 | 역할 |
|---|---|
| Network Namespace | Pod의 네트워크 환경 격리 |
| veth pair | Pod Namespace와 Host 연결 |
| Linux Bridge | MAC 주소를 기준으로 Frame 전달 |
| Routing | IP 주소를 기준으로 next hop 결정 |
| ARP | Next hop IP에 해당하는 MAC 확인 |
| Netfilter | Packet 처리 지점 제공 |
| iptables | Netfilter 규칙 설정 |
| CNI | Pod의 실제 네트워크 구성 |

> Network Namespace가 Pod 네트워크를 격리하고, veth가 Pod와 Host를 연결한다. Bridge 또는 Routing이 패킷 경로를 만들고, ARP가 다음 장치의 MAC 주소를 찾는다. Netfilter와 iptables는 그 경로에서 패킷을 허용하거나 변환한다.

---

## 11. 학습하면서 생긴 질문

### Q1. 모든 CNI가 veth pair와 Linux Bridge를 사용하는 것은 아니라면, Calico나 Cilium의 패킷 경로는 어떻게 다를까?

### Q2. 다른 Node에 있는 Pod로 패킷을 보낼 때 Pod IP는 유지하면서 실제 Node 사이에서는 어떻게 패킷을 전달할까?

---

## 12. 참고 자료

- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Kubernetes Network Model](https://kubernetes.io/docs/concepts/services-networking/)
- [CNI Specification](https://github.com/containernetworking/cni/blob/main/SPEC.md)
- [Network Namespace와 veth — Linux man-pages](https://man7.org/linux/man-pages/man7/network_namespaces.7.html)
- [Linux Bridge — Linux Kernel Documentation](https://docs.kernel.org/networking/bridge.html)
- [iptables — Linux man-pages](https://man7.org/linux/man-pages/man8/iptables.8.html)
