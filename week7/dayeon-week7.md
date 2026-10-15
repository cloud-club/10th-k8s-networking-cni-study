# Week 7. eBPF Deep Dive & Hubble

## 1. eBPF Program 실행 과정

### 1.1 eBPF Program은 어떻게 Kernel에서 실행되는가?

eBPF Program은 Kernel의 특정 Hook에 연결되어 네트워크 패킷이나 시스템 이벤트를 처리하는 프로그램입니다.

사용자가 작성한 코드가 그대로 실행되는 것이 아니라, Kernel에 로드되고 검증된 뒤 실행 가능한 형태로 준비됩니다.

```text
eBPF Source Code
       │
       ▼
Compiler (예: Clang/LLVM)
       │
       ▼
eBPF Bytecode
       │
       ▼
BPF System Call
       │
       ▼
Kernel Verifier
       │
       ├── 검증 실패 → Load 거부
       │
       ▼
JIT Compiler (지원 시)
       │
       ▼
Machine Code
       │
       ▼
Hook에 Attach
       │
       ▼
Packet / Event 발생 시 실행
```

- **Compiler**: C 등의 소스 코드를 eBPF Bytecode로 변환함.
- **Verifier**: 메모리 접근, 포인터 사용, 프로그램의 종료 가능성 등 안전성 관련 조건을 검사함.
- **JIT (Just-In-Time Compiler)**: eBPF 명령어를 CPU가 실행할 수 있는 Machine Code로 변환함. JIT 사용 여부는 Kernel 설정 등에 따라 달라짐.
- **Hook**: 프로그램이 실행될 Kernel 내부 지점임. 네트워크에서는 XDP, TC ingress/egress 등이 사용됨.
- **Attach**: eBPF Program을 특정 Hook에 연결하는 과정임.

Verifier는 프로그램이 안전하게 실행될 수 있는지 검증함. 모든 네트워크 동작이 올바른지 검증하는 것은 아니므로, 프로그램이 로드되었다고 해서 네트워크 정책이나 라우팅이 의도대로 동작한다는 의미는 아님.

참고: [Linux Kernel — eBPF Verifier](https://docs.kernel.org/bpf/verifier.html)

### 1.2 Cilium에서의 실행 과정

Cilium Agent는 Endpoint 및 정책 등의 상태를 관리하고, 필요한 eBPF Program과 Map을 준비하여 Kernel Datapath를 구성합니다.

```text
Kubernetes API
      │
      ▼
  Cilium Agent
      │
      ├── Endpoint / Identity 관리
      ├── Policy 처리
      ├── BPF Map 갱신
      └── eBPF Program 준비 및 연결
                  │
                  ▼
             Linux Kernel
                  │
                  ▼
          Packet 처리 시 실행
```

중요한 차이:

- **Cilium Agent**: 정책과 네트워크 상태를 관리하고 Datapath를 구성함.
- **eBPF Program**: 패킷이 Hook에 도달했을 때 Kernel 안에서 실제 처리를 수행함.

따라서 일반적인 패킷 처리마다 Cilium Agent가 직접 개입하는 구조는 아님.

## 2. BPF Map과 Cilium Datapath

### 2.1 BPF Map의 역할

BPF Map은 eBPF Program과 사용자 공간 프로세스 사이에서 네트워크 상태 및 처리 정보를 공유하기 위한 Kernel 자료구조임.

eBPF Program이 패킷을 처리할 때마다 모든 정책이나 Service 정보를 새로 계산하는 대신, 미리 구성된 Map을 조회해 판단할 수 있음.

```text
             Cilium Agent
                  │
                  │ Map 갱신
                  ▼
             ┌──────────┐
             │ BPF Maps │
             └────┬─────┘
                  ▲
                  │ Lookup / Update
                  │
             eBPF Program
                  ▲
                  │
                Packet
```

- **Agent**: Kubernetes 리소스 및 네트워크 상태의 변화에 따라 Map을 갱신함.
- **BPF Map**: Datapath가 참조할 정보를 저장함.
- **eBPF Program**: 패킷 처리 중 필요한 Map을 조회하여 전달, 차단, 변환 등의 동작을 결정함.

Map의 내용과 구조는 종류마다 다르며, 모든 Map이 동일한 방식으로 갱신되거나 조회되는 것은 아님.

### 2.2 주요 BPF Map

| Map | 역할 |
|---|---|
| IP Cache (`ipcache`) | IP 주소와 보안 Identity, 네트워크 경로 관련 정보를 연결하는 데 사용 |
| Service LB Map | Service IP 및 포트와 Backend 정보를 이용한 Service 처리에 사용 |
| Policy Map | Endpoint에 적용할 허용 규칙 등을 조회하는 데 사용 |
| Connection Tracking Map | 연결 상태를 추적하고 기존 연결의 패킷을 처리하는 데 사용 |
| NAT Map | 주소 변환 상태를 관리하는 데 사용 |

각 Map은 서로 다른 목적을 가짐. 예를 들어 Service LB Map은 목적지 Service를 Backend로 변환하는 데 사용되고, Policy Map은 해당 트래픽이 허용되는지 판단하는 데 사용됨.

참고: [Cilium — eBPF Maps](https://docs.cilium.io/en/stable/network/ebpf/maps/)

### 2.3 Packet 처리와 Map 조회

예를 들어 Pod A가 Kubernetes Service의 ClusterIP로 요청을 보내는 경우를 생각해 봅니다.

```text
Pod A
  │
  │ 목적지: Service ClusterIP
  ▼
eBPF Program
  │
  ├── Service LB Map 조회
  │       │
  │       └── Service → Backend 매핑 확인
  │
  ├── Policy 관련 정보 확인
  │       │
  │       └── 허용 여부 판단
  │
  ▼
Backend Pod로 전달
```

위 그림은 개념적인 흐름임. 실제 처리 순서와 사용되는 Hook은 Cilium 설정, 트래픽 방향, Service 유형 등에 따라 달라질 수 있음.

핵심은 **Agent가 관리하는 네트워크 상태를 BPF Map에 저장하고, eBPF Program이 패킷 처리 시 이를 참조한다는 것**임.

### 2.4 ipcache와 Identity

Cilium은 IP 주소뿐 아니라 Endpoint의 보안 Identity를 활용해 정책을 적용함.

```text
Pod A
  │
  ▼
IP: 10.0.1.10
  │
  ▼
Identity: app=frontend
  │
  ▼
Policy 판단
  │
  ├── 허용
  └── 차단
```

`ipcache`는 IP 주소와 Identity 등의 정보를 연결하는 데 사용됨. 이를 통해 IP 주소만을 기준으로 하는 것이 아니라, Cilium이 관리하는 Identity 및 정책 정보를 활용할 수 있음.

다만 IP Cache가 모든 NetworkPolicy 규칙을 직접 저장하는 것은 아님. Identity 관련 정보와 실제 정책 판단에 사용되는 Policy Map은 구분해야 함.

## 3. eBPF Datapath와 NetworkPolicy

### 3.1 정책은 어떻게 적용되는가?

Kubernetes에서 NetworkPolicy를 생성하면 Cilium Agent가 정책을 처리하고, 필요한 Datapath 상태를 구성함.

```text
NetworkPolicy
      │
      ▼
Cilium Agent
      │
      ▼
Policy / Identity 정보 갱신
      │
      ▼
BPF Map 및 eBPF Datapath
      │
      ▼
Packet 도착
      │
      ▼
Policy 판단
      │
      ├── FORWARDED
      └── DROPPED
```

이때 Policy Map은 실제 패킷 처리에 필요한 허용 정보를 제공함. 정책이 변경되면 Agent가 관련 상태를 갱신하고, 이후 패킷은 갱신된 Datapath 상태에 따라 처리됨.

### 3.2 통신이 차단되는 경우

예를 들어 Frontend Pod에서 Backend Pod로 TCP 8080 연결을 시도한다고 가정합니다.

```text
Frontend Pod
     │
     │ TCP / 8080
     ▼
eBPF Policy 처리
     │
     ▼
Backend 접근 정책 확인
     │
     ├── 허용 규칙 일치 → 전달
     │
     └── 허용 규칙 없음 → 차단
```

NetworkPolicy가 적용된 환경에서 통신이 실패하면 다음을 구분해서 확인해야 함.

- Pod가 올바른 IP를 가지고 있는가?
- 목적지로 향하는 경로가 존재하는가?
- Service가 올바른 Backend를 가리키는가?
- NetworkPolicy가 해당 방향의 트래픽을 허용하는가?
- 실제로 패킷이 전달되었는가, 차단되었는가?

**통신 실패를 무조건 NetworkPolicy 문제로 단정하지 않고, 주소·경로·Service·정책을 분리해 확인하는 것이 중요함.**

## 4. Hubble: eBPF Event에서 Flow까지

### 4.1 Hubble은 무엇을 보여주는가?

Hubble은 Cilium Datapath의 네트워크 관측 기능으로, Endpoint 사이의 통신과 정책 처리 결과 등을 Flow 형태로 확인할 수 있게 함.

Hubble 자체가 패킷을 전달하거나 NetworkPolicy를 직접 적용하는 것은 아님. 실제 패킷 처리는 Cilium Datapath가 수행하고, Hubble은 그 과정에서 생성되는 관측 정보를 활용함.

```text
Packet
  │
  ▼
Cilium eBPF Datapath
  │
  ├── 전달 / 차단
  │
  └── 관측 이벤트 생성
            │
            ▼
      Cilium Agent
            │
            ▼
      Hubble Server
            │
            ▼
      Hubble CLI / UI
            │
            ▼
       Flow 분석
```

위 구조는 개념적인 흐름임. 이벤트 수집 및 전달의 세부 경로는 구성에 따라 달라질 수 있음.

### 4.2 Flow에서 확인할 수 있는 정보

Hubble Flow에서는 다음과 같은 정보를 확인할 수 있음.

- Source / Destination Endpoint
- Source / Destination IP
- Protocol 및 Port
- 트래픽 방향
- 처리 결과 및 Drop Reason
- 관련 Identity 및 Policy 정보(해당 이벤트에 제공되는 경우)

예시:

```text
SOURCE       DESTINATION      PROTOCOL   VERDICT
frontend     backend:8080     TCP        FORWARDED
frontend     backend:8080     TCP        DROPPED
```

- `FORWARDED`: 해당 관측 지점에서 패킷이 전달 처리되었음을 의미함.
- `DROPPED`: 해당 관측 지점에서 패킷이 차단되었음을 의미함.

단, `FORWARDED`가 애플리케이션 요청의 성공이나 응답 수신까지 보장하지는 않음. 패킷이 전달되었더라도 목적지 애플리케이션이 응답하지 않을 수 있음.

또한 Flow 이벤트는 관측 지점과 이벤트 유형에 따라 의미가 다를 수 있으므로, 하나의 Flow만 보고 전체 통신 경로를 단정하지 않아야 함.

## 5. Hubble을 이용한 Troubleshooting

### 5.1 기본 명령어

Cilium CLI가 설치된 환경에서는 다음 명령어를 활용할 수 있음.

```bash
# 실시간 Flow 관찰
hubble observe

# Drop 이벤트 확인
hubble observe --verdict DROPPED

# 특정 Pod 관련 Flow 확인
hubble observe --pod <namespace>/<pod-name>

# 특정 IP 관련 Flow 확인
hubble observe --ip <ip-address>
```

명령어와 필터 지원 여부는 설치된 Hubble 버전 및 실행 환경에 따라 다를 수 있음. 먼저 `hubble observe --help`로 확인하는 것이 좋음.

### 5.2 사례 1: Pod 간 통신 실패

상황: Frontend Pod에서 Backend Pod의 TCP 8080 포트로 요청했지만 응답이 없음.

**Step 1. Flow 확인**

```bash
hubble observe --verdict DROPPED
```

관련 Drop 이벤트가 있는지 확인함.

**Step 2. Source / Destination 확인**

- Source가 예상한 Frontend Pod인지 확인함.
- Destination IP와 포트가 예상한 Backend인지 확인함.
- 다른 Pod나 다른 포트로 요청이 향하고 있지 않은지 확인함.

**Step 3. Drop Reason 확인**

정책 위반으로 차단되었는지, 다른 원인으로 처리되지 않았는지 확인함. 구체적인 사유는 이벤트에 제공되는 정보를 기준으로 판단함.

**Step 4. NetworkPolicy 확인**

```bash
kubectl get networkpolicy -A
kubectl describe networkpolicy <policy-name> -n <namespace>
```

- Ingress / Egress 방향을 구분함.
- Pod selector와 namespace selector를 확인함.
- TCP 8080 포트가 허용되는지 확인함.

**Step 5. Drop 이벤트가 없다면**

Hubble에서 관련 이벤트를 찾지 못했다고 해서 패킷이 정상 전달되었다고 단정할 수 없음. 필터, 관측 범위, 이벤트 수집 상태를 확인하고 다음 항목을 점검함.

- Pod IP 및 라우팅
- Service와 EndpointSlice
- Backend 애플리케이션의 Listening Port
- TCP 연결 및 응답 여부

### 5.3 사례 2: Service 통신 문제

상황: Pod IP로 직접 요청하면 성공하지만 ClusterIP로 요청하면 실패함.

```text
Pod A
  │
  ├── Pod IP 직접 요청 → 성공
  │
  └── ClusterIP 요청 → 실패
```

이 경우 NetworkPolicy만 확인하지 말고 Service 처리 경로도 확인해야 함.

- Service의 ClusterIP 및 Port가 올바른가?
- EndpointSlice에 정상 Backend가 등록되어 있는가?
- Hubble Flow에서 실제 목적지가 예상한 Backend로 나타나는가?
- Drop 이벤트가 발생하는가?
- Service LB Map 및 Cilium Agent 상태에 이상이 있는가?

Hubble은 실제 관측된 Flow를 통해 패킷의 출발지, 목적지, 처리 결과를 확인하는 데 도움을 줌. 다만 내부 Map의 정확한 상태까지 모든 Flow가 직접 보여주는 것은 아니므로, 필요하면 Cilium 상태 및 BPF Map 관련 진단 도구를 함께 사용함.

## 6. 핵심 정리

### eBPF 실행

- eBPF Program은 Kernel에 로드되기 전에 Verifier의 안전성 검증을 거침.
- JIT는 eBPF 명령어를 Native Machine Code로 변환할 수 있음.
- Program은 Hook에 연결되어 관련 패킷이나 이벤트가 발생할 때 실행됨.

### Cilium Datapath

- Cilium Agent는 네트워크 상태와 정책을 관리하고 Datapath를 구성함.
- BPF Map은 IP Cache, Service, Policy, Connection Tracking 등의 정보를 저장함.
- eBPF Program은 패킷 처리 중 필요한 Map을 조회하여 전달 또는 차단 등을 결정함.

### Hubble

- Hubble은 Cilium Datapath에서 발생하는 네트워크 관측 정보를 Flow 형태로 제공함.
- `FORWARDED`는 애플리케이션 통신 성공과 동일하지 않음.
- `DROPPED` Flow와 Drop Reason은 정책 및 통신 문제 분석에 유용함.
- 관련 Flow가 없을 때는 관측 범위와 수집 상태까지 확인해야 함.

## Cilium certificate 

### 예상 문제 1. eBPF와 iptables의 차이

[ Question 1 ]

A Kubernetes cluster uses Cilium for networking. Which statement best describes a key benefit of eBPF-based networking compared with traditional iptables-based Service handling?

A. eBPF eliminates the Linux Kernel from the packet forwarding path.

B. eBPF can perform packet processing in the Kernel using programs and maps, reducing reliance on large sets of iptables rules for supported features.

C. eBPF requires every Pod to run a sidecar proxy.

D. eBPF guarantees that all iptables rules are removed from the Node.

<details>
<summary>정답 및 해설</summary>

**정답: B**

eBPF Program은 Kernel의 Hook에 연결되어 패킷을 직접 처리하고, BPF Map을 조회해 Service 및 정책 관련 정보를 사용할 수 있음. 이를 통해 지원되는 기능은 기존 iptables 기반 처리에 대한 의존도를 줄일 수 있음.

- A: eBPF는 오히려 Kernel 안에서 실행됨.
- C: eBPF 네트워킹은 모든 Pod에 sidecar를 요구하지 않음.
- D: Cilium에서도 환경과 기능에 따라 iptables 규칙이 남을 수 있음.

</details>

## 예상 문제 2. BPF Map의 역할

[ Question 2 ]

A packet is destined for a Kubernetes Service ClusterIP. In an eBPF-based datapath, what is the primary purpose of a Service load-balancing BPF Map?

A. To store the source code of eBPF programs before compilation.

B. To store Kubernetes YAML manifests and NetworkPolicy objects directly.

C. To provide Service-to-Backend information that an eBPF program can look up during packet processing.

D. To collect all packet payloads and display them in Hubble.

<details>
<summary>정답 및 해설</summary>

**정답: C**

Service LB Map은 Service 주소·포트와 Backend 정보를 연결하는 데 사용됨. eBPF Program은 패킷 처리 중 Map을 조회해 적절한 Backend를 선택하거나 목적지 변환에 필요한 정보를 얻을 수 있음.

- A: eBPF Program은 별도의 컴파일·로드 과정을 거침.
- B: Kubernetes 객체 자체를 YAML 형태로 저장하는 용도가 아님. Agent가 Kubernetes 상태를 해석하고 Datapath에 필요한 정보를 반영함.
- D: BPF Map은 Hubble의 패킷 캡처 저장소가 아님. Hubble은 Datapath에서 발생하는 관측 이벤트를 Flow로 제공함.

</details>

## 예상 문제 3. Hubble Flow 해석

[ Question 3 ]

Hubble reports a `FORWARDED` flow from a frontend Pod to a backend Pod on TCP port 8080. However, the application request times out. What is the best next step?

A. Assume the backend application responded successfully because the flow verdict is FORWARDED.

B. Immediately disable all NetworkPolicies in the cluster.

C. Verify the observed source/destination and port, then inspect the backend listener, return traffic, and relevant routing or policy state.

D. Reinstall Cilium because FORWARDED indicates that the eBPF program failed.

<details>
<summary>정답 및 해설</summary>

**정답: C**

`FORWARDED`는 해당 관측 지점에서 패킷이 전달 처리되었다는 뜻이지, 애플리케이션이 요청을 처리하고 응답까지 반환했다는 뜻은 아님.

다음 순서로 범위를 좁히는 것이 적절함.

1. Flow의 Source, Destination, TCP 포트가 예상과 일치하는지 확인함.
2. Backend 애플리케이션이 해당 포트에서 Listening 중인지 확인함.
3. 응답 패킷이 돌아오는지 확인하고, 필요하면 양방향 Flow를 살펴봄.
4. 관련 라우팅, Service/Endpoint, NetworkPolicy를 점검함.

Hubble은 네트워크 흐름을 추적하는 관측 도구이며, 애플리케이션의 정상 처리까지 보장하는 도구는 아님.

</details>

## 참고 자료

- [eBPF 기반의 강력한 쿠버네티스 네트워킹: Cilium CNI 소개 : kt cloud 기술 블로그](https://tech.ktcloud.com/entry/eBPF-%EA%B8%B0%EB%B0%98%EC%9D%98-%EA%B0%95%EB%A0%A5%ED%95%9C-%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%82%B9-Cilium-CNI-%EC%86%8C%EA%B0%9C)
- [kube-proxy가 느린 이유와 Cilium이 해결하는 방법](https://youtu.be/4miCVaIgd08?si=lPseWc0ZWiPKfNSo)
- [Cilium — eBPF Datapath](https://docs.cilium.io/en/stable/network/ebpf/)
- [Cilium — eBPF Maps](https://docs.cilium.io/en/stable/network/ebpf/maps/)
- [Cilium — Hubble](https://docs.cilium.io/en/stable/observability/hubble/)
- [Linux Kernel — eBPF Verifier](https://docs.kernel.org/bpf/verifier.html)
