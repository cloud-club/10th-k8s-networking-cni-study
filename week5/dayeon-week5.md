
Cilium = eBPF를 활용하여 Kubernetes 네트워킹, Service Load Balancing, NetworkPolicy, Observability 등을 Linux Kernel 수준에서 처리하는 CNI

## 1. Cilium Architecture

### Cilium의 주요 구성요소

- **Cilium Agent**
  - 각 Node에 DaemonSet으로 실행됨
  - Kubernetes API를 감시하고 네트워크 정책, Endpoint 정보를 관리
  - eBPF 프로그램을 Kernel에 로드

- **eBPF**
  - Linux Kernel 내부의 특정 Hook 지점에서 프로그램을 실행
  - Packet 처리, Load Balancing, NetworkPolicy enforcement 등에 사용

- **Hubble**
  - Cilium의 Observability 계층
  - eBPF가 수집한 네트워크 이벤트를 기반으로 통신 흐름과 정책 동작을 관찰


## 2. eBPF 기본 개념

eBPF(extended Berkeley Packet Filter)는 Linux Kernel 내부에서 안전하게 프로그램을 실행할 수 있는 기술입니다.

기존에는 Kernel 기능을 변경하려면 Kernel Module 등을 사용해야 했지만, eBPF를 사용하면 필요한 로직을 특정 Hook에 동적으로 연결할 수 있습니다.

```
Application
     ↓
Network Stack
     ↓
┌──────────────────────┐
│ Linux Kernel         │
│                      │
│   eBPF Program       │
│      ↓               │
│ Packet 처리          │
│ Policy 검사          │
│ Load Balancing       │
└──────────────────────┘
```

### 주요 Hook

네트워크에서는 대표적으로 다음과 같은 위치에 eBPF를 붙일 수 있음.

* **XDP**

  * NIC에 매우 가까운 위치
  * 매우 빠른 Packet 처리 가능
  * DDoS filtering, Packet drop 등에 활용

* **TC (Traffic Control)**

  * Linux Network Interface의 ingress/egress traffic 처리
  * Cilium의 주요 datapath에서 활용

* **Socket / cgroup Hook**

  * Socket 및 Application 수준의 네트워크 동작 제어

## 3. Cilium Datapath

기존 Kubernetes 네트워킹에서는 여러 Kernel 기능을 조합해서 Packet을 처리함.

```
Pod
 ↓
veth
 ↓
iptables / IPVS
 ↓
Routing
 ↓
Network Interface
```

Cilium은 eBPF를 활용하여 이러한 Packet 처리 과정의 일부를 Kernel 내부에서 직접 수행함.

```
Pod
 ↓
veth
 ↓
eBPF
 ├─ Routing
 ├─ NetworkPolicy
 ├─ Load Balancing
 └─ Connection Tracking
 ↓
Network
```

즉, **eBPF 자체가 CNI를 대체하는 것이 아니라 Cilium이 eBPF를 사용하여 Kernel datapath를 구성하는 것**이 핵심임.

## 4. kube-proxy Replacement

기존 Kubernetes Service의 기본적인 구조:

```
Client
  ↓
Service IP
  ↓
kube-proxy
  ↓
iptables / IPVS
  ↓
Pod
```

Cilium은 eBPF를 이용하여 Service Load Balancing을 Kernel에서 직접 처리할 수 있음.

```
Client
  ↓
Service IP
  ↓
eBPF
  ↓
Backend Pod
```

따라서 kube-proxy가 관리하던 iptables/IPVS 기반 Service forwarding을 eBPF 기반으로 대체할 수 있음.

**주의:** `kube-proxy replacement`는 단순히 kube-proxy 프로세스를 삭제하는 것이 아니라, kube-proxy가 담당하던 기능을 Cilium/eBPF datapath가 대신 수행한다는 의미임.

## 5. NetworkPolicy와 Identity

기존 IP 기반 NetworkPolicy:

```
Source IP → Destination IP
             ↓
          Allow / Deny
```

Cilium은 Kubernetes identity 및 eBPF를 활용하여 보다 효율적으로 정책을 적용할 수 있음.

```
Pod
 ↓
Identity
 ↓
eBPF Policy
 ↓
Allow / Drop
```

예를 들어:

```
frontend
   ↓
   X  → database
```

처럼 Pod의 IP 자체보다 **어떤 Kubernetes Endpoint/Identity에서 온 traffic인지**를 기준으로 정책을 적용할 수 있음.

## 6. Hubble

Hubble은 Cilium의 Observability 시스템

eBPF가 실제 datapath에서 처리하는 정보를 활용하여:

```
Pod A
 ↓
HTTP Request
 ↓
Pod B
 ↓
Policy Allow
```

같은 Network Flow를 관찰할 수 있음.

따라서 단순히:

```bash
kubectl get networkpolicy
```

로 정책 객체가 존재하는지만 확인하는 것이 아니라,

```
Traffic
   ↓
eBPF
   ↓
Policy decision
   ↓
Hubble
   ↓
Flow / Drop / Verdict 확인
```

처럼 **실제 datapath에서 traffic이 어떻게 처리됐는지 확인할 수 있음.**

## 7. 기존 CNI와 Cilium의 차이

| 항목             | 기존 CNI                           | Cilium                        |
| -------------- | -------------------------------- | ----------------------------- |
| Pod Networking | CNI + Linux routing/iptables 등   | CNI + eBPF                    |
| Service        | kube-proxy + iptables/IPVS       | eBPF 가능                       |
| Policy         | iptables 등 활용                    | eBPF                          |
| Identity       | 주로 IP 기반                         | Kubernetes Identity 활용        |
| Observability  | 별도 도구 필요                         | Hubble                        |
| Packet 처리      | Kernel networking stack 여러 계층 활용 | eBPF를 활용해 Kernel datapath 최적화 |

## 핵심 정리

### eBPF를 어디에 사용하는가?

```text
                  Linux Kernel
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
       XDP             TC       Socket/cgroup
        │              │              │
     Packet         Packet         Socket
     처리           처리           처리
                       ↑
                    Cilium
```
