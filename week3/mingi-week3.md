# CNI Overview & Cloud Native Networking Landscape - week 3

## CNI(Container Network Interface)

**Linux 컨테이너의 네트워크 인터페이스를 구성하기 위한 플러그인을 작성하는 명세와 라이브러리, 그리고 지원되는 여러 플러그인으로 이루어진 표준 인터페이스**

- CNCF(Cloud Native Computing Foundation)에서 관리
- 주로 Kubernetes, containerd, CRI-O 등의 컨테이너 런타임 환경에서 사용
- 컨테이너가 생성될 때 IP 할당, 가상 네트워크 구성, 라우팅 설정 등을 자동으로 처리

**CNI 대표 동작**

| 명령 | 의미 |
| --- | --- |
| `ADD` | Pod 네트워크 생성, 인터페이스/IP/라우트 설정 |
| `DEL` | Pod 삭제 시 네트워크 자원 정리 |
| `CHECK` | 기존 네트워크 설정이 정상인지 검사 |
| `VERSION` | 지원하는 CNI 규격 버전 확인 |

**CNI Plugin**

| 플러그인 | 분류 | 역할 |
| --- | --- | --- |
| `loopback` | 범용 CNI Plugin | Pod 내부 `lo` 인터페이스 구성 |
| `bridge` | 범용 CNI Plugin | Linux bridge에 Pod veth 연결 |
| `host-local` | 범용 IPAM Plugin | 노드 로컬 범위에서 Pod IP 할당 |
| `dhcp` | 범용 IPAM Plugin | DHCP 서버에서 Pod IP 할당 |
| `portmap` | 범용 Meta Plugin | `hostPort`를 위한 포트 매핑 |
| `bandwidth` | 범용 Meta Plugin | Pod ingress/egress 대역폭 제한 |
| `flannel` | Primary CNI Plugin | Flannel 노드 대역 정보를 바탕으로 Pod 네트워크 구성 |
| `calico` | Primary CNI Plugin | Pod 인터페이스 연결, Calico IPAM·정책 시스템 연동 |
| `calico-ipam` | IPAM Plugin | Calico IP Pool에서 Pod IP 할당 |
| `cilium-cni` | Primary CNI Plugin | Pod veth를 Cilium eBPF 데이터 패스에 연결 |

### IPAM(IP Address Management)

Pod에게 고유한 IP를 할당하고, 중복 없이 관리하는 시스템

- 역할
    - 사용 가능한 IP 탐색
    - Pod에 IP 할당
    - 중복 할당 방지
    - 할당 상태 저장
    - Pod 삭제 시 IP 회수
    - 필요하면 게이트웨이와 라우트 정보 반환
- CIDR 범위

| CNI | 흔히 사용하는 Pod CIDR |
| --- | --- |
| Flannel | `10.244.0.0/16` |
| Calico | `192.168.0.0/16` |
| Cilium | `10.0.0.0/8` |
| Cilium 노드별 CIDR | `/24` |
- IPAM 방식

| IPAM 방식 | 주소를 가져오는 곳 | 특징 |
| --- | --- | --- |
| `host-local` | 노드별 로컬 CIDR | 단순하고 빠르지만 노드별 CIDR 분리가 필요 |
| `static` | CNI 설정에 지정된 IP | 고정 주소, 일반 Pod 대규모 관리에는 부적합 |
| `dhcp` | 외부 DHCP 서버 | 기존 네트워크의 DHCP 이용 |
|  | Calico IPAM | Calico IPPool |
| Cilium IPAM | Cilium Pool 또는 Node CIDR | 다양한 IPAM 모드 지원 |
| Cloud IPAM | AWS VPC, Azure VNet 등 | 클라우드 네트워크 IP를 Pod에 직접 할당 |

#### Kubernetes가 노드별 Pod CIDR을 할당하는 방식

1. `kube-controller-manager`가 노드에 Pod CIDR 블록 할당 (ex. `Node1` → `10.0.1.0/24`)
2. `CNI IPAM` 이 개별 Pod에 IP 할당 (ex. `Pod1` → `10.0.1.2`)

```bash
Cluster Pod CIDR
10.0.0.0/16
      │
      │ kube-controller-manager
      │ Node IPAM Controller
      ▼
Node 1: 10.0.1.0/24
Node 2: 10.0.2.0/24
      │
      │ CNI IPAM
      ▼
Pod IP (Pod 1: 10.0.1.2)
```

#### CNI가 자체 IP Pool을 관리하는 방식

- 역할

```bash
kube-controller-manager
  └── Node CIDR 할당 비활성화 또는 미사용

Calico/Cilium 등의 IPAM
  ├── 전체 IP Pool 관리
  ├── 노드에 IP 블록 분배
  ├── Pod IP 할당
  └── Pod 삭제 시 IP 회수
```

- 동작

```bash
CNI IP Pool
10.0.0.0/16
      │
      │ CNI의 자체 IPAM
      ▼
Node 1용 IP Block
10.0.1.0/26

Node 2용 IP Block
10.0.2.0/26
      │
      ▼
개별 Pod IP 할당
```

### Primary CNI

**클러스터의 모든 일반 Pod에 제공할 기본 네트워크를 담당하는 CNI 솔루션**

- 역할
    - Pod 네트워크 네임스페이스에 `eth0` 생성
    - 호스트와 Pod를 `veth` 쌍으로 연결
    - IPAM을 통해 Pod IP 할당
    - Pod의 기본 Gateway와 Route 설정
    - 같은 노드 및 다른 노드 Pod까지 갈 수 있는 경로 구성
    - 제품에 따라 NetworkPolicy, 암호화, 관측성 등을 추가 제공

#### 대표적인 Primary CNI

| Primary CNI | 기본 강점 |
| --- | --- |
| Flannel | 단순하게 Pod 간 연결성 제공 |
| Calico | L3 라우팅, BGP, NetworkPolicy |
| Cilium | eBPF 기반 네트워킹·정책·관측성 |
| AWS VPC CNI | Pod에 AWS VPC 네트워크 자원과 IP 연결 |
| Azure CNI | Pod를 Azure 네트워크와 통합 |

Flannel·Calico·Cilium은 “CNI Plugin 하나”라기보다 다음을 묶은 **Primary CNI 솔루션**이다.

```
Primary CNI 솔루션
├─ CNI Plugin: Pod 생성 시 eth0, veth 연결
├─ IPAM: Pod IP 할당
├─ Node Agent: 노드별 라우팅·정책·데이터 패스 관리
└─ Control Plane: IP Pool, 정책, 클러스터 정보 관리
```

#### Secondary CNI

Primary CNI가 기본 네트워크를 만들고, Secondary CNI은 그 기능을 덧붙이는 형태

```
Primary CNI: Calico
├─ 기본 eth0 / Pod IP / Pod 간 통신
├─ portmap: 선택, hostPort 제공
├─ bandwidth: 선택, 대역폭 제한
└─ tuning: 선택, MTU·sysctl 등 조정
```

#### Secondary CNI와 비교

| 구분 | Primary CNI | Secondary CNI |
| --- | --- | --- |
| 대상 인터페이스 | 기본 `eth0` | 추가 `net1`, `net2` 등 |
| 적용 대상 | 일반적으로 모든 Pod | 필요한 특정 Pod |
| 목적 | Kubernetes 기본 Pod 네트워크 | 특수 네트워크 연결 |
| 예시 | Calico, Cilium, Flannel | macvlan, ipvlan, SR-IOV |
| 대표 조합 | Cilium | Multus + SR-IOV |

**ex. Multus**

```bash
eth0  → Calico/Cilium 기본 클러스터 네트워크
net1  → SR-IOV 고성능 네트워크
```

## Flannel

**Kubernetes 클러스터에서 Pod 간의 네트워크 통신을 가능하게 해주는 CNI 플러그인**

- 특징
    - **오버레이 네트워크(Overlay Network):** 기본적으로 **VXLAN** 방식을 사용하여 파드 간의 원래 패킷을 UDP 패킷으로 한 번 더 캡슐화하여 노드 간 통신을 수행 (MTU 감소, 성능 오버헤드)
 
    <img width="800" alt="image" src="https://github.com/user-attachments/assets/9288920f-a60c-4239-b9de-a0d8a7744fcb" />
    
    - **단순한 구조:** Pod-to-Pod 연결성에 집중해 이해와 설치가 비교적 쉬움
    - **IP 할당:** 각 노드에 고정된 서브넷(CIDR 대역)을 할당하여 노드 내 파드들이 고유한 IP를 가지도록 관리
    - **NetworkPolicy 지원 X**
    - **CIDR 범위:** `10.244.0.0/16`

## Calico

**컨테이너, 가상 머신 및 기본 호스트 기반 워크로드를 위한 오픈 소스 네트워킹 및 네트워크 보안 솔루션**

- 특징
    - **L3 순수 IP 라우팅**: 오버레이 네트워크 없이 라우팅 테이블을 직접 사용하여 성능이 우수
    (Pod를 L3 라우팅 가능한 엔드포인트로 본다)
    - **BGP 프로토콜 지원**: `BIRD` 데몬을 통해 물리 라우터와 직접 연동되어 대규모 클러스터 확장에 유리
        - BGP 프로토콜: 스위치나 라우터 등의 네트워크 장비에서 통신하기 위한 상대의 ip 대역을 전파하기 위해 사용되는 프로토콜
    - **고급 네트워크 정책(Network Policy)**: 기본 쿠버네티스 정책보다 세밀한 트래픽 제어, Global Network Policy, 계층(Tier) 관리 등을 지원
    - **다양한 데이터 플레인**: 표준 Linux 네트워킹 외에도 고성능 처리를 위한 **eBPF 모드**를 지원
    - **CIDR 범위:** `192.168.0.0/16`
- 구성요소

    <img width="800" alt="image" src="https://github.com/user-attachments/assets/5a56a586-0f04-449c-b570-3b70fbc0839f" />

    - Felix (필릭스) : 인터페이스 관리, 라우팅 정보 관리, ACL 관리, 상태 체크
    - BIRD (버드): BGP Peer 에 라우팅 정보 전파 및 수신, BGP RR(Route Reflector)
    - Confd : calico global 설정과 BGP 설정 변경 시(트리거) BIRD 에 적용해줌
    - Datastore plugin : calico 설정 정보를 저장하는 곳 - k8s API datastore(kdd) 혹은 etcd 중 선택
    - Calico IPAM plugin : 클러스터 내에서 파드에 할당할 IP 대역
    - calico-kube-controllers : calico 동작 관련 감시(watch)
    - calicoctl : calico 오브젝트를 CRUD 할 수 있다, 즉 datastore 접근 가능
- 전달 방식

| 방식 | 동작 |
| --- | --- |
| Native Routing / BGP | Pod CIDR 경로를 직접 라우팅. 캡슐화 없음 |
| IP-in-IP | Pod 패킷을 IP 패킷으로 한 번 감싸 노드 간 전달 |
| VXLAN | Pod 패킷을 UDP/VXLAN으로 감싸 노드 간 전달 |
| eBPF Dataplane | eBPF로 패킷 전달·정책·Service 처리를 수행하는 선택 구성 |

## Cilium

**리눅스 커널 레벨에서 동작하는 eBPF 기술을 기반으로 쿠버네티스 네트워킹과 보안, 가시성을 제공하는 고성능 CNI 솔루션**

`eBPF` : 리눅스 커널에서 작은 사용자 정의 코드를 실행할 수 있게 해주는 기술

<img width="773" height="275" alt="image" src="https://github.com/user-attachments/assets/43a3e5d4-7e88-4844-9c46-ae0382f35a99" />

- 특징
    - **고성능 네트워킹**: 전통적인 `iptables` 방식의 오버헤드를 제거하고 커널 내에서 패킷을 직접 처리하여 네이티브 수준의 속도를 제공
    - **Kube-proxy 대체**: `kube-proxy` 없이 eBPF 기반으로 효율적인 로드 밸런싱을 수행해 대규모 클러스터에서도 부하를 크게 줄여준다
    - **세밀한 보안 정책**: L3, L4뿐만 아니라 HTTP, gRPC 등 L7 계층까지 신원(Identity) 기반의 강력하고 정교한 네트워크 정책 적용이 가능
    - **뛰어난 가시성(Hubble)**: 패킷 손실이나 성능 저하 없이 실시간으로 네트워크 흐름과 트래픽을 모니터링

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

### Kube-proxy Replacement

Cilium은 선택 설정으로 `kube-proxy`를 대체 가능

기존에는 `kube-proxy`가 iptables/IPVS 규칙으로 Service 트래픽을 실제 Pod로 전달했다면, Cilium은 eBPF Map에서 Service와 Backend Pod 정보를 조회해 로드밸런싱과 NAT를 처리함.

```
Client
→ Service ClusterIP
→ Cilium eBPF Service Map
→ Backend Pod 선택
```

### Hubble

**Cilium의 eBPF 데이터 패스를 기반으로 한 네트워크 관측 도구**

- 역할
    - 어떤 Pod가 어느 Pod 또는 외부 IP와 통신했는지 확인
    - 통신이 정책에 의해 허용되었는지 또는 차단되었는지 확인
    - HTTP, DNS 등 지원되는 L7 트래픽 정보 확인 가능
    - `Hubble Relay`: 여러 노드의 흐름 정보를 모아 조회
    - `Hubble UI`: 네트워크 흐름과 Service Map을 시각화

- 참고 자료
    
    https://github.com/projectcalico/calico/blob/master/cni-plugin/pkg/k8s/k8s.go
    
    https://github.com/projectcalico/calico/blob/master/cni-plugin/pkg/plugin/plugin.go
    
    https://github.com/aws/amazon-ecs-cni-plugins/tree/master/plugins
    
    https://www.floodnut.com/127
    
    https://captcha.tistory.com/78
    
    https://github.com/flannel-io/flannel
    
    https://docs.tigera.io/calico/latest/about/
    
    https://hellouz818.tistory.com/77
