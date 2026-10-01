# Cilium & eBPF - week 5

| 항목 | Flannel | Calico | Cilium |
| --- | --- | --- | --- |
| 기본 철학 | 단순한 Pod Connectivity | L3 Routing + Security | eBPF 기반 Networking + Security |
| 대표 Datapath | Linux Bridge + VXLAN | Linux Routing | eBPF + Linux Networking |
| Overlay | VXLAN | VXLAN / IPIP | VXLAN / Geneve |
| Native Routing | 제한적/host-gw | 강력한 지원 | 지원 |
| BGP | X | 지원 | 지원 가능 |
| NetworkPolicy | flanneld 자체 핵심 기능 아님 | 강력하게 지원 | 강력하게 지원 |
| Policy 구현 | 별도 정책 구성 필요 | iptables/nftables/eBPF 등 | eBPF |
| IPAM | Node subnet 중심 | Calico IPAM | Cilium IPAM |
| 구조 | 단순 | 중간~복잡 | 상대적으로 복잡 |
| 주요 특징 | 간단한 Overlay | Routing/BGP | eBPF/Identity |

---

## Cilium Architecture

**리눅스 커널 레벨에서 동작하는 eBPF 기술을 기반으로 쿠버네티스 네트워킹과 보안, 가시성을 제공하는 고성능 CNI 솔루션**

<p align="center"><img width="600" alt="image" src="https://github.com/user-attachments/assets/375ebd9c-f7e6-4e7a-a241-9282007b40e5" />


- 구성 요소

| 구성요소 | 역할 |
| --- | --- |
| **cilium-cni** | Pod 생성·삭제 시 컨테이너 런타임이 호출하는 플러그인으로, 노드의 Agent와 협력해 Pod 네트워크 구성 |
| **cilium-agent** | 각 노드에서 실행되며 Pod·Service·정책 변경을 감시하고 eBPF 프로그램과 Map 관리 |
| **cilium-operator** | IPAM 관련 작업 등 클러스터 전체에서 관리해야 하는 작업 수행 |
| **eBPF Datapath** | 노드의 Linux 커널에서 패킷 전달, Service 처리, 정책 검사 수행 |
| **Envoy** | HTTP 등 애플리케이션 계층의 정책 검사에 사용하는 프록시 |
| **Hubble** | Datapath와 프록시에서 발생한 통신 이벤트를 수집·조회 |

### 동작 흐름

**설정 및 정책 관리 흐름**

1. 사용자가 Kubernetes API를 통해 Service와 네트워크 정책 등을 정의
2. Cilium Operator가 IPAM 관련 작업 등 클러스터 범위의 리소스 관리
3. 각 노드의 Cilium Agent가 Pod·Service·정책 변경 감지
4. Agent가 Endpoint와 Identity를 관리하고 필요한 정보를 eBPF Map에 반영
5. Agent가 커널 Hook에 연결한 eBPF 프로그램이 해당 정보를 조회하여 트래픽 처리

Operator의 작업과 Agent의 변경 감시는 **병렬적인 역할**이다. 정책이 반드시 `API → Operator → Agent` 순서로 전달되는 것은 아니다.

**패킷 및 연결 처리 흐름**

1. 애플리케이션의 소켓 동작 또는 네트워크 인터페이스의 패킷 수신·송신 발생
2. 해당 Hook의 eBPF 프로그램 실행
3. Map을 조회하여 Service backend 선택, 정책 검사, 목적지 변환·전달 등 수행
4. 정책상 허용된 트래픽은 목적지로 전달하고, 차단 대상은 폐기
5. L7 정책 검사가 필요하면 Envoy 등의 프록시를 통해 추가 검사
6. 처리 과정에서 설정에 따라 관찰 이벤트 발생

Service 처리는 패킷이 인터페이스에 도착하기 전 **socket Hook**에서도 수행할 수 있다.

**모니터링 흐름**

1. 각 노드의 Hubble Server가 Datapath와 프록시의 관찰 이벤트 수집
2. Hubble Relay가 여러 노드의 흐름을 클러스터 범위에서 조회하도록 제공
3. Hubble CLI로 이벤트를 조회하거나 Hubble UI로 통신 관계 시각화
4. 운영자가 전달·차단 결과를 확인하여 통신 문제와 정책 적용 상태 분석

---

### eBPF와 hook

#### **eBPF(extended Berkeley Packet Filter)**

리눅스 커널에서 작은 사용자 정의 코드를 실행할 수 있게 해주는 기술

<img width="773" height="275" alt="image" src="https://github.com/user-attachments/assets/2dfa66ae-b43f-4b57-83be-0d105db3dec1" />


- 장점

| 장점 | 설명 |
| --- | --- |
| **효율적인 처리** | 필요한 작업을 커널 안에서 수행하므로, 사용자 공간 프로그램으로 데이터를 넘겨 처리하는 비용을 줄일 수 있음 |
| **이른 시점의 처리** | 전체 네트워크 스택을 거치기 전에 처리하여 후속 작업을 줄일 수 있음 |
| **유연한 기능 추가** | 커널 소스를 수정하거나 별도 커널 모듈을 작성하지 않고 프로그램을 적재·연결할 수 있음 |
| **상태 갱신의 유연성** | 프로그램이 조회하는 Map 데이터를 갱신하여 변경된 상태를 반영할 수 있음 |
| **다양한 관찰 지점** | 패킷, 소켓, 커널 함수 등 여러 이벤트에서 정보를 얻을 수 있음 |
| **실행 전 안전성 검사** | Verifier가 메모리 접근과 실행 경로 등을 검사하여 위험한 프로그램의 적재를 제한 |

**커널의 필요한 지점에서 직접 코드를 실행하여 네트워크 처리와 관찰 기능을 유연하게 구현할 수 있다**

#### **eBPF Map**

eBPF 프로그램이 처리에 필요한 정보를 저장하고 조회하는 커널 내부의 데이터 구조

| 정보 | 예시 |
| --- | --- |
| **Service·backend 정보** | Service 주소 → 실제 backend Pod 목록 |
| **Identity 정보** | Pod IP → Security Identity |
| **정책 정보** | 특정 Identity의 TCP 8080 접근 허용 |
| **연결 상태** | 이미 처리 중인 연결의 상태 |

예를 들어 Service로 연결 요청이 들어오면, **eBPF 프로그램이 Map에서 backend 정보를 조회하여 실제 목적지를 선택**한다.

사용자 공간의 Cilium Agent와 커널의 eBPF 프로그램 모두 Map에 접근할 수 있다. Agent가 변경된 정보를 반영하면 프로그램이 이를 이용해 트래픽을 처리한다.

**eBPF 프로그램이 ‘처리하는 코드’라면, Map은 ‘그 코드가 사용하는 데이터’이다.**

#### **hook**

**프로그램이 실행되는 위치·이벤트**

| Hook | 실행 지점 | 주요 용도 |
| --- | --- | --- |
| **XDP** | 네트워크 드라이버 최하단에 위치(패킷 수신 직후 가장 먼저 실행) | 빠른 패킷 필터링, 선택적인 NodePort·LoadBalancer 가속 |
| **TC ingress/egress** | 네트워크 인터페이스의 수신·송신 지점 | L3/L4 정책 검사, 패킷 변환·전달 |
| **cgroup socket Hook** | `connect()` 등 애플리케이션의 소켓 동작 지점 | Service backend 선택, 소켓 목적지 주소 변경 |

**XDP** : *패킷 처리*

- 네트워크 드라이버 최하단에 위치*(패킷 수신 직후 가장 먼저 실행)*.
- **커널 네트워크 스택 이전에 네트워크 카드 드라이버 레벨에서 직접 런타임 프로그램 실행**.
- 패킷 처리 성능 최상

**TC Ingress/Egress** : *패킷 처리, 정책 적용, 트래픽 redirection*

- 네트워크 인터페이스에 부착*(초기 패킷 처리 이후 L3 레이어 진입 직전 실행)*.
- 패킷과 연관된 metadata에 접근 가능.
- 로컬 노드의
    - **패킷 처리**
    - **L3/L4 정책 적용**
    - **트래픽 redirection**
    - **veth pair의 Host side에 부착되어 컨테이너를 출입하는 모든 트래픽 감시, 정책 강제**에 적합

**sock_ops** : *소켓 레이어 가속화*

- 특정 cgroup에 부착*(TCP 상태 전환 이벤트(`ESTABLISHED`) 발생 시 실행)*.
- (연결된 peer가 동일 로컬 노드/로컬 프록시일 때) **소켓 레이어 가속화** 수행.

**sockmap** : *소켓 통신*

- TCP 소켓 전송(send) 작업 수행 시 실행.
- 메시지 검사 후 drop, 일반 TCP 계층으로 전송 가능
- **네트워크 스택 거치지 않고 다른 소켓으로 메시지 직접 redirect 가능**
- 소켓 통신 가속화

**한 줄 정리: Hook의 이벤트가 발생하면, 그 Hook에 연결된 eBPF 프로그램이 실행되는 것**

<p align="center"><img width="600" alt="image" src="https://github.com/user-attachments/assets/759d850f-fb92-4d38-8e7e-85b8d7ee4b1a" />


Cilium에서의 Datapath는 다음과 같이 **조합**된다.

- **eBPF Datapath + Overlay:** eBPF로 트래픽을 처리하고, 노드 간에는 캡슐화하여 전달
- **eBPF Datapath + Native Routing:** eBPF로 트래픽을 처리하고, 노드 간에는 Pod IP 라우팅으로 전달

---

### Kube-proxy replacement

Cilium은 선택 설정으로 `kube-proxy`를 대체 가능

기존에는 `kube-proxy`가 iptables/IPVS 규칙으로 Service 트래픽을 실제 Pod로 전달했다면, Cilium에서는Cilium Agent가 정보를 준비하고, eBPF 프로그램이 Map을 조회하여 backend 선택과 목적지 변환 등을 수행한다.

| 역할 | kube-proxy 사용 | Cilium Replacement |
| --- | --- | --- |
| Service·backend 변경 감시 | kube-proxy | Cilium Agent |
| 처리 정보 관리 | iptables·nftables 등의 규칙 | eBPF Service·backend Map 등 |
| 실제 트래픽 처리 | 커널의 해당 네트워크 기능 | 커널 Hook의 eBPF 프로그램 |

---

### Identity 기반 정책

일반적인 IP 기반 정책은 “`10.0.1.10`에서 오는 트래픽을 허용한다”처럼 **Pod IP 주소**를 기준으로 규칙을 만든다.

하지만 Kubernetes Pod는 재시작·재배포되면 IP가 바뀔 수 있다.

```
기존 frontend Pod
- Label: app=frontend
- IP: 10.0.1.10

Pod 재생성 후
- Label: app=frontend
- IP: 10.0.1.27
```

IP만 기준으로 정책을 적용하면 Pod의 IP가 바뀔 때마다 정책을 다시 맞춰야 할 수 있다.

Cilium은 Pod IP 대신 Label을 보고, 같은 Label 조합을 가진 Pod에 하나의 **Security Identity**를 부여한다.

```
Label: app=frontend, role=api
→ Security Identity: 1234
```

그리고 정책도 “Identity 1234, 즉 `app=frontend`인 Pod만 backend에 접근 가능”처럼 적용한다.

```
app=frontend Pod
→ backend Pod:8080 허용

그 외 Pod
→ backend Pod:8080 차단
```

따라서 frontend Pod가 재생성되어 IP가 `10.0.1.10`에서 `10.0.1.27`로 바뀌어도, `app=frontend` Label이 유지되면 같은 Identity를 사용하므로 기존 정책이 그대로 적용된다.

> 즉, **Cilium Identity는 IP처럼 바뀔 수 있는 주소가 아니라, Pod의 역할을 나타내는 Label을 기반으로 정책을 적용하기 위한 식별자이다.**
> 

---

### Hubble

**Cilium의 eBPF 데이터 패스를 기반으로 한 네트워크 관측 도구**

- 구성 요소
    
    
    | 구성요소 | 역할 |
    | --- | --- |
    | **Hubble Server** | 각 노드의 Cilium Agent 내부에서 동작하며 해당 노드의 통신 이벤트 제공 |
    | **Hubble Relay** | 여러 노드의 Hubble Server에 연결하여 클러스터 범위의 조회 제공 |
    | **Hubble CLI** | 명령줄에서 통신 이벤트 조회·필터링 |
    | **Hubble UI** | 서비스 사이의 연결 관계를 그래프로 시각화 |

**각 노드에서 통신 이벤트를 수집하고, Relay를 통해 클러스터 전체의 흐름을 조회하는 구조**

<p align="center"><img width="250" alt="image" src="https://github.com/user-attachments/assets/377f5644-481e-4c6b-9b45-66eee6da7cbc" />


- 역할
    - 어떤 Pod가 어느 Pod 또는 외부 IP와 통신했는지 확인
    - 통신이 정책에 의해 허용되었는지 또는 차단되었는지 확인
    - HTTP, DNS 등 지원되는 L7 트래픽 정보 확인 가능
    - `Hubble Relay` 를 통해 여러 노드의 흐름 정보를 모아 조회
    - `Hubble UI`를 통해 네트워크 흐름과 Service Map을 시각화

---

- 참고 자료
    
    https://docs.cilium.io/en/stable/overview/component-overview/
    
    https://velog.io/@daankwak/Kubernetes-Cilium-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B8%B0
    
    https://velog.io/@baeyuna97/Cilium
    
    https://ygtoken.tistory.com/387
