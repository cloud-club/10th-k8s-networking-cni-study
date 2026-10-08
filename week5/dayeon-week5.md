
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

## 2. eBPF란?

eBPF(extended Berkeley Packet Filter)는 **Linux Kernel 내부에서 특정 이벤트가 발생했을 때 실행할 수 있는 프로그램을 동적으로 로드하는 기술**임.

원래 BPF는 Packet Filtering을 목적으로 만들어졌지만, 현재 eBPF는 Networking뿐 아니라 Observability, Security, Tracing 등 다양한 Kernel 기능에 사용됨.

Cilium에서는 eBPF를 활용하여 Packet 처리, Load Balancing, NetworkPolicy, Connection Tracking 등을 Kernel 내부에서 수행함.

### 왜 eBPF가 필요한가?

기존에는 네트워크 기능을 구현하기 위해 주로 다음과 같은 Kernel 기능을 조합했음.

```
Packet
  ↓
iptables / IPVS
  ↓
Routing
  ↓
Network Stack
```

eBPF를 사용하면 필요한 로직을 Kernel의 특정 Hook에 직접 연결할 수 있음.

```
Packet
  ↓
┌─────────────────────┐
│ Linux Kernel        │
│                     │
│ eBPF Program        │
│  ├─ Policy          │
│  ├─ Load Balancing  │
│  ├─ Routing         │
│  └─ Observability   │
└─────────────────────┘
  ↓
Network
```

즉, **eBPF는 새로운 네트워크 프로토콜이나 CNI가 아니라 Kernel의 동작을 프로그래밍할 수 있게 해주는 메커니즘**임.

## 3. eBPF는 어떻게 동작하는가?

eBPF 프로그램은 일반적으로 다음 과정을 거쳐 Kernel에 로드됨.

```
C / eBPF Program
       ↓
      LLVM
       ↓
  eBPF Bytecode
       ↓
  bpf() syscall
       ↓
 Kernel Verifier
       ↓
      JIT
       ↓
Kernel에 프로그램 등록
       ↓
특정 Hook에서 실행
```

### 1) Compile

C 등의 코드로 작성한 프로그램을 LLVM 등을 이용해 eBPF bytecode로 변환함.

### 2) Load

User Space에서 `bpf()` system call 등을 통해 프로그램을 Kernel에 전달함.

### 3) Verifier

Kernel의 **Verifier**가 프로그램이 안전하게 실행될 수 있는지 검사함.

예:

* 허용되지 않은 메모리 접근 여부
* 잘못된 Pointer 사용 여부
* 무한 실행 가능성 여부

### 4) JIT

검증된 eBPF bytecode는 JIT(Just-In-Time Compiler)를 통해 CPU가 직접 실행할 수 있는 Native Instruction으로 변환될 수 있음.

따라서 User Space로 Packet을 넘겨 별도의 프로그램에서 처리하는 방식보다 Kernel 내부에서 빠르게 처리할 수 있음.

## 4. eBPF Architecture

eBPF는 단순히 "프로그램 하나를 Kernel에서 실행한다"로 끝나지 않음.

핵심 구성요소는 다음과 같음.

```
                 User Space
                     │
              Cilium Agent
                     │
              bpf() syscall
                     │
                     ▼
              ┌─────────────┐
              │   Kernel    │
              │             │
              │  Verifier   │
              │      ↓      │
              │    eBPF     │
              │   Program   │
              │      │      │
              │ ┌────┴────┐ │
              │ ↓         ↓ │
              │ Maps    Helpers
              │             │
              └──────┬──────┘
                     │
                  Hook Point
                     │
              ┌──────┴──────┐
              ↓             ↓
             XDP            TC
              │             │
              └──────┬──────┘
                     ↓
                  Packet
```

### eBPF Program

실제 로직을 수행하는 프로그램.

예:

* Packet Drop
* NAT
* Load Balancing
* Policy 검사

### eBPF Maps

Kernel에 저장되는 **Key-Value 형태의 데이터 구조**.

eBPF Program끼리 상태를 공유하거나 User Space의 Cilium Agent와 데이터를 공유할 때 사용함.

Cilium에서는 Endpoint, Service, NAT, Connection Tracking 등의 상태를 BPF Map에 저장하여 datapath에서 활용함. 

```
Cilium Agent
     │
     ↓
 BPF Map 업데이트
     │
     ↓
eBPF Program
     │
     ↓
Packet 처리
```

### Helper

eBPF Program이 Kernel의 기능을 사용할 수 있도록 제공되는 함수.

예를 들어 Packet redirect, Map 접근 등의 작업에 사용함.

### Tail Call

하나의 eBPF Program에서 다른 eBPF Program으로 실행을 넘길 수 있는 기능.

Cilium처럼 복잡한 datapath를 여러 단계로 나누어 구성할 때 활용할 수 있음. 

## 5. eBPF Hook

eBPF는 아무 위치에서나 실행되는 것이 아니라 **Kernel이 제공하는 Hook Point에 attach**됨.

네트워크에서 중요한 것은 다음 두 가지임.

### XDP

Network Driver에 매우 가까운 위치에서 실행됨.

```text
NIC
 ↓
XDP
 ↓
Linux Network Stack
```

가장 빠르게 Packet을 검사하거나 Drop할 수 있기 때문에 고성능 Filtering 등에 적합함.

### TC

XDP보다 Linux Network Stack 안쪽에서 실행됨.

```text
NIC
 ↓
XDP
 ↓
Network Stack
 ↓
TC ingress/egress
 ↓
Routing
```

XDP보다 더 많은 Kernel networking 정보와 기능을 활용할 수 있어 Cilium의 주요 datapath에 사용됨.

## 6. Cilium에서는 eBPF가 어떻게 연결되는가?

Cilium Agent가 Kubernetes의 상태를 관찰하고 필요한 네트워크 정보를 eBPF Program과 Map에 반영함.

```
Kubernetes API
      ↓
Cilium Agent
      │
      ├── Endpoint / Identity
      ├── Service
      ├── NetworkPolicy
      │
      ↓
eBPF Program + BPF Maps
      ↓
Linux Kernel Hook
      ↓
Packet
      ↓
Policy / LB / Routing / NAT
      ↓
Destination
```

따라서 Cilium에서 중요한 관계는:

> **Cilium Agent = Control / Configuration**
> **eBPF = Kernel Datapath**
> **BPF Map = Control Plane과 Datapath가 상태를 공유하는 공간**

이라고 이해하면 됨.

## 7. eBPF vs iptables

기존 Kubernetes networking:

```
Packet
  ↓
iptables rules
  ↓
Rule matching
  ↓
Routing / NAT
```

Cilium eBPF datapath:

```
Packet
  ↓
eBPF Hook
  ↓
BPF Map lookup
  ↓
Policy / LB / Routing
  ↓
Next Hop
```

핵심 차이는 **eBPF가 iptables보다 무조건 빠르다**가 아니라,

* 필요한 로직을 Kernel datapath에 직접 프로그래밍할 수 있음
* BPF Map을 이용해 상태를 효율적으로 조회할 수 있음
* Service LB, Policy, Observability 등을 하나의 datapath에서 통합할 수 있음
* 프로그램을 동적으로 교체/업데이트할 수 있음

이라는 점임.


## 핵심 정리

```text
eBPF
 └─ Linux Kernel을 Programmable하게 만드는 기술
       │
       ├─ Program
       │    └─ 실제 처리 로직
       │
       ├─ Map
       │    └─ 상태 / 정보 저장
       │
       ├─ Helper
       │    └─ Kernel 기능 사용
       │
       └─ Hook
            ├─ XDP
            ├─ TC
            └─ Socket / cgroup
```

**Cilium은 이 eBPF 기능을 이용하여** Network, Load Balancing, NetworkPolicy, Routing, Observability를 Linux Kernel datapath에서 처리함.

> **한 줄 요약:**
> eBPF는 Linux Kernel에 안전하게 프로그램을 동적으로 연결할 수 있게 해주는 기술이고, Cilium은 이를 네트워크 datapath에 활용하여 기존 iptables 중심의 처리를 eBPF 기반으로 구현함.


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

## 8. Cilium Datapath

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

## 9. kube-proxy Replacement

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

## 10. NetworkPolicy와 Identity

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

## 11. Hubble

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

## 12. 기존 CNI와 Cilium의 차이

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

Ref:
- [1]: https://training.linuxfoundation.org/certification/cilium-certified-associate-cca/?utm_source=chatgpt.com "Cilium Certified Associate (CCA) - Linux Foundation - Education"
- [2]: https://docs.cilium.io/en/latest/reference-guides/bpf/architecture/?utm_source=chatgpt.com "BPF Architecture — Cilium 1.21.0-dev documentation"
- [3]: https://docs.cilium.io/en/latest/network/ebpf/maps/?utm_source=chatgpt.com "eBPF Maps — Cilium 1.21.0-dev documentation"
- [4]: https://docs.cilium.io/en/stable/network/ebpf/intro/?utm_source=chatgpt.com "Introduction — Cilium 1.20.2 documentation"
