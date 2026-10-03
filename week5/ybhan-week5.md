# W5. Cilium & eBPF

## 1. W4에서 이어지는 질문

이번 주에는 세 가지를 보려 합니다.

- eBPF가 무엇이고, 커널의 어디에 붙어서 패킷을 처리하는지
- Cilium이 Service 처리와 정책을 iptables 대신 무엇으로 하는지
- IP 대신 Identity로 정책을 판단하면 무엇이 달라지는지

이번 주차 정리도 전부 Cilium 공식 문서와 eBPF 프로젝트 문서(ebpf.io)를 기준으로 했습니다.



## 2. 용어 정리


| 용어                     | 정의                                     | 역할                                  |
| ---------------------- | -------------------------------------- | ----------------------------------- |
| eBPF                   | 커널 같은 특권 영역에서 샌드박스된 프로그램을 실행하는 기술      | 커널을 고치지 않고 커널 동작에 로직을 끼워 넣음          |
| Hook                   | eBPF 프로그램이 붙는 미리 정해진 지점                | 시스템 콜, 커널 트레이스포인트, 네트워크 이벤트 등        |
| Verifier               | eBPF 프로그램을 로드하기 전에 안전성을 검사하는 단계        | 권한 확인, 시스템을 망가뜨리지 않는지, 반드시 끝나는지 확인   |
| JIT                    | 바이트코드를 CPU별 기계어로 번역                    | 실행 속도 최적화                           |
| eBPF Map               | eBPF 프로그램이 데이터를 저장하고 꺼내는 자료구조          | 해시 테이블, 배열, LRU, 링 버퍼, LPM 등         |
| XDP                    | eXpress Data Path, 네트워크 드라이버의 가장 이른 Hook | 네트워크 스택 처리 전에 패킷을 다룸                |
| Endpoint               | IP 주소를 공유하는 애플리케이션 컨테이너 묶음              | Kubernetes에서는 Pod 하나가 Endpoint 하나    |
| Identity               | 보안 관련 레이블로 정해지는 클러스터 전체 고유 식별자         | 정책 판단의 기준. 같은 레이블의 Pod끼리 공유          |
| Envoy                  | Cilium이 L7 정책에 쓰는 노드 로컬 프록시              | HTTP, DNS 같은 애플리케이션 계층 규칙 적용          |


참고자료: [What is eBPF (ebpf.io)](https://ebpf.io/what-is-ebpf/), [Cilium 용어](https://docs.cilium.io/en/stable/gettingstarted/terminology/)



## 3. eBPF 기본 개념과 Hook

### 3.1 eBPF가 동작하는 방식

ebpf.io는 eBPF를 운영체제 커널 같은 특권 영역에서 샌드박스된 프로그램을 실행할 수 있게 하는 기술로 정의합니다. 커널 소스를 고치거나 커널 모듈을 넣지 않고도 커널 동작 중간에 로직을 끼워 넣을 수 있다는 뜻.

프로그램이 커널에 들어가기까지:

```text
eBPF 프로그램(바이트코드)
    → Verifier: 권한이 있는지 / 시스템을 망가뜨리지 않는지 / 반드시 끝나는지(무한 루프 X)
    → JIT: CPU별 기계어로 번역
    → Hook에 부착: 이벤트가 생길 때마다 실행
```

실행 중에 쓰는 도구는 세 가지입니다.


| 도구          | 내용                                     |
| ----------- | -------------------------------------- |
| Map         | 프로그램끼리, 또는 사용자 공간과 데이터를 주고받는 자료구조       |
| Helper call | 커널이 제공하는 안정적인 API 함수 (시간 조회, 패킷 조작 등)  |
| Tail call   | 다른 eBPF 프로그램으로 실행을 넘김. `execve()`처럼 현재 실행 맥락을 교체 |


Verifier가 없으면 잘못 짠 프로그램 하나가 커널 전체를 멈출 수 있습니다. 커널 안에서 돌리면서도 안전하다고 말할 수 있는 근거가 이 검사 단계입니다.

참고자료: [What is eBPF](https://ebpf.io/what-is-ebpf/)

### 3.2 Cilium이 쓰는 네트워크 Hook


| Hook                  | 위치                                  | Cilium의 쓰임                        |
| --------------------- | ----------------------------------- | --------------------------------- |
| XDP                   | 네트워크 드라이버의 가장 이른 지점, 스택 처리 전         | 패킷 수신 즉시 프로그램 실행                  |
| tc ingress / egress   | 스택이 초기 처리를 한 뒤, L3 처리 전              | 노드 로컬 정책 적용, 트래픽을 Endpoint로 리다이렉트 |
| Socket operations     | 특정 cgroup에 붙어서 TCP 이벤트마다 실행           | TCP 상태 변화 관찰 (특히 ESTABLISHED 진입)  |
| Socket send / recv    | TCP 소켓의 모든 send 동작마다 실행              | 소켓 계층에서 메시지 검사, drop, 리다이렉트       |
| L7 Policy (eBPF 밖)    | 사용자 공간 프록시                          | Envoy로 트래픽을 넘겨 L7 정책 적용           |


W1에서 그렸던 Netfilter 흐름(PREROUTING → 경로 판단 → INPUT/FORWARD)과 비교하면, XDP는 문서 표현 그대로 "다른 어떤 네트워크 스택 처리보다 먼저" 실행됩니다. 소켓 계층 Hook은 반대로 스택 아래쪽이 아니라 소켓 계층에서 동작합니다. 6절의 kube-proxy 대체가 이 소켓 Hook을 씁니다.

참고자료: [Cilium eBPF Datapath 소개](https://docs.cilium.io/en/stable/network/ebpf/intro/)

### 3.3 eBPF Map의 한도

iptables에서 규칙이 순서대로 쌓였다면, Cilium은 상태를 Map에 둡니다. Map은 크기가 정해져 있어서 기본 한도가 문서에 나와 있습니다.


| Map                   | 범위       | 기본 한도              |
| --------------------- | -------- | ------------------ |
| Connection Tracking   | 노드       | TCP 512k / UDP 256k |
| NAT                   | 노드       | 512k               |
| Neighbor Table        | 노드       | 512k               |
| Endpoints             | 노드       | 64k                |
| IP cache              | 노드       | 512k               |
| Service Load Balancer | 노드       | 64k                |
| Policy                | Endpoint | 16k                |


`--bpf-map-dynamic-size-ratio`로 전체 메모리 대비 비율을 정해 큰 Map(Connection Tracking, NAT, Service LB 등)의 상한을 조정할 수 있습니다. 예를 들어 0.0025면 시스템 메모리의 0.25%.

한도의 단위는 항목 수입니다. Service LB Map의 64k는 Service 개수가 아니라, 문서 기준으로 대략 다음처럼 계산되는 항목 수입니다.

```text
LB Map 항목 수 ≈ Service 수 × Service당 평균 백엔드 수 × Service당 평균 포트/프로토콜 수
```

모든 Map은 한도를 넘으면 더 이상 항목이 들어가지 않습니다. 그리고 노드에서 Cilium agent가 처음 뜰 때 Map이 만들어지기 때문에, 나중에 크기를 바꾸고 재시작하면 Map을 다시 채우는 동안 연결이 끊깁니다. 문서는 설치 전에 크기를 미리 따져보라고 권합니다.

참고자료: [Cilium eBPF Maps](https://docs.cilium.io/en/stable/network/ebpf/maps/)



## 4. Cilium Architecture

### 4.1 구성 요소


| 구성                      | 위치        | 역할                                                    |
| ----------------------- | --------- | ----------------------------------------------------- |
| cilium-agent            | 노드마다      | 커널이 컨테이너 출입 트래픽을 통제하는 데 쓰는 eBPF 프로그램을 관리               |
| cilium-operator         | 클러스터 단위   | 클러스터 전체에서 한 번만 처리하면 되는 작업 담당                         |
| cilium-cni              | 노드의 CNI 바이너리 | Pod가 스케줄되거나 종료될 때 Kubernetes가 호출. Pod 네트워크와 정책 구성     |
| cilium-dbg              | agent와 함께 설치 | 같은 노드 agent의 REST API와 통신하는 디버그 CLI                 |
| Hubble server           | 노드마다      | eBPF 기반 가시성을 받아 gRPC로 flow와 Prometheus 메트릭 제공         |
| Hubble Relay            | 클러스터 단위   | 모든 Hubble server를 알고 클러스터 전체 가시성 제공                  |
| Hubble CLI / UI         | 사용자 쪽     | flow 조회 / 서비스 의존성·연결 맵 시각화                           |
| 데이터스토어                  | 클러스터      | 기본은 Kubernetes CRD, 규모를 위해 etcd를 선택적으로 사용            |


W3에서 본 노드별 에이전트 + 조정 컨트롤러 구조가 Cilium에서도 그대로입니다. agent가 노드 에이전트, operator가 조정 컨트롤러 자리.

참고자료: [Cilium 컴포넌트 개요](https://docs.cilium.io/en/stable/overview/component-overview/)

### 4.2 operator나 agent가 없으면

문서는 operator가 잠시 내려가면 IPAM이 지연될 수 있고, etcd kvstore를 쓰는 경우 heartbeat가 갱신되지 않아 agent가 재시작될 수 있다고 설명합니다. 클러스터 단위 작업이 멈추는 것.

agent 쪽은 시스템 요구사항 문서에 단서가 있습니다. eBPF 파일시스템을 `/sys/fs/bpf`에 마운트해두면 **agent가 재시작돼도 eBPF 자원이 유지되어, 업그레이드 중에도 datapath가 끊기지 않는다**고 합니다. W4 질문 4("에이전트가 죽으면 이미 커널에 들어간 경로로 기존 통신은 계속되는가")에 대해 Cilium은 이 방식으로 답하고 있는 셈입니다. W2에서 정리한 규칙을 넣는 쪽과 패킷을 옮기는 쪽을 나눈 구조가 여기서도 보입니다.

참고자료: [Cilium 컴포넌트 개요](https://docs.cilium.io/en/stable/overview/component-overview/), [Cilium 시스템 요구사항](https://docs.cilium.io/en/stable/operations/system_requirements/)

### 4.3 요구사항

- Linux 커널 5.10 이상 또는 동등한 커널 (예: RHEL 8.10의 4.18)
- `CONFIG_BPF`, `CONFIG_BPF_SYSCALL`, `CONFIG_NET_CLS_BPF`, `CONFIG_BPF_JIT` 등 eBPF 커널 옵션
- 노드 사이에 열어야 하는 포트


| 포트        | 용도               |
| --------- | ---------------- |
| UDP 8472  | VXLAN 오버레이        |
| UDP 6081  | Geneve 오버레이       |
| TCP 4240  | cilium-health    |
| TCP 4244  | Hubble server    |
| TCP 4245  | Hubble Relay     |
| UDP 51871 | WireGuard 암호화    |


W4에서 Flannel VXLAN이 커널 기본값 8472를 쓴다고 정리했는데, Cilium VXLAN도 같은 8472입니다.

참고자료: [Cilium 시스템 요구사항](https://docs.cilium.io/en/stable/operations/system_requirements/)



## 5. Cilium Datapath

### 5.1 라우팅 모드

W4에서 본 오버레이 vs underlay 구분이 Cilium에서는 두 모드로 나옵니다.


| 모드              | 설정                         | 조건                                              | 비용 / 비고                        |
| --------------- | -------------------------- | ----------------------------------------------- | ------------------------------ |
| Encapsulation (기본) | VXLAN(UDP 8472) 또는 Geneve(UDP 6081) | 노드끼리 IP/UDP로 닿기만 하면 됨. 하부 네트워크가 PodCIDR을 몰라도 됨 | VXLAN은 패킷당 50바이트 MTU 오버헤드. 점보 프레임으로 상당 부분 상쇄 |
| Native Routing  | `routing-mode: native` + `ipv4-native-routing-cidr` | 하부 네트워크가 Pod 주소로 IP 포워딩을 할 수 있어야 함           | 경로 배포는 `auto-direct-node-routes: true` 또는 BGP |


기본값이 캡슐화인 이유를 문서는 "하부 네트워크 인프라에 대한 요구사항이 가장 적은 모드"라고 설명합니다. W4에서 Flannel 기본값이 VXLAN이었던 이유와 같습니다.

문서가 꼽는 캡슐화 모드의 장점 중에 오버레이 공통 장점 외에 Cilium만의 것이 하나 있습니다. **캡슐화 헤더에 Identity 같은 메타데이터를 실어 보낼 수 있다**는 것. 7절 Identity 기반 정책과 이어지는 부분입니다.

클라우드에서는 AWS ENI 모드(VPC IP를 Pod에 직접 할당해 캡슐화가 없는 대신 인스턴스당 Pod 수가 ENI 한도에 묶임), GKE의 Alias IP 기반 native routing이 있습니다.

참고자료: [Cilium 라우팅](https://docs.cilium.io/en/stable/network/concepts/routing/)

### 5.2 패킷 흐름

문서의 Life of a Packet은 세 가지 흐름으로 나눕니다.


| 흐름                 | 내용                                                              |
| ------------------ | --------------------------------------------------------------- |
| 같은 노드 Endpoint 간   | 송신·수신 양쪽에 선택적으로 L7 정책                                            |
| Endpoint → 밖(다른 노드) | 오버레이면 오버레이용 인터페이스(보통 `cilium_vxlan`)로 나감. 암호화를 켜면 이 단계에서 암호화    |
| 밖 → Endpoint       | 오버레이 해제, 암호화된 패킷은 먼저 복호화 후 일반 흐름으로                             |


소켓 계층 정책 적용을 켜면 TCP 연결이 ESTABLISHED가 된 뒤에는 Endpoint 정책을 매번 다시 거치지 않고 L7 정책 객체만 거칩니다. 연결 단위로 한 번 판단하고 나면 이후 패킷은 가볍게 지나가는 구조.

참고자료: [Life of a Packet](https://docs.cilium.io/en/stable/network/ebpf/lifeofapacket/)

### 5.3 eBPF Host-Routing: iptables를 건너뛰기

튜닝 문서에 따르면 eBPF Host-Routing을 켜면 Cilium은 **iptables와 호스트 상위 스택을 완전히 우회하고**, 일반 veth보다 빠르게 네트워크 네임스페이스를 전환합니다. 조건은 eBPF 기반 kube-proxy 대체와 eBPF 기반 masquerading이 둘 다 켜져 있는 것.

W4에서 비워둔 eBPF 칸을 이 내용으로 채우면:


| 방식   | 장점                                       | 비용                                  |
| ---- | ---------------------------------------- | ----------------------------------- |
| eBPF | Host-Routing으로 iptables와 호스트 상위 스택을 우회 가능 | 커널 5.10 이상, eBPF 커널 옵션 필요. 기능별로 요구 커널이 더 높아짐 (netkit 6.8+, IPv4 BIG TCP 6.3+) |


참고자료: [Cilium 성능 튜닝](https://docs.cilium.io/en/stable/operations/performance/tuning/)

### 5.4 IPAM

Cilium IPAM 모드는 Kubernetes Host Scope, Cluster Scope(기본값), Multi-Pool, CRD-backed, AWS ENI, Azure IPAM, GKE 일곱 가지입니다.

운영 중에 IPAM 모드를 바꾸면 기존 워크로드 연결이 계속 끊길 수 있다고 문서가 경고합니다. 온라인으로 지원하는 전환은 cluster-pool → multi-pool 하나뿐이고, 나머지는 새 클러스터를 만들라고 권합니다.

참고자료: [Cilium IPAM](https://docs.cilium.io/en/stable/network/concepts/ipam/)



## 6. kube-proxy Replacement

### 6.1 무엇을 대체하나

`kubeProxyReplacement=true`로 배포하면 Cilium이 kube-proxy를 완전히 대체합니다. 대상은 ClusterIP, NodePort, LoadBalancer, externalIPs Service와 hostPort.

W3 질문 2("원래 iptables 규칙이 하던 일은 어디로 옮겨가는가")의 답은 문서 기준으로 두 군데입니다.

- Service와 백엔드 정보 → eBPF Map (3.3절 Service Load Balancer Map, 노드당 기본 64k)
- 목적지 주소 변환 시점 → 패킷 단위가 아니라 소켓 단위

### 6.2 소켓 단위 부하분산 (socket-LB)

문서 설명 그대로, `connect`(TCP, connected UDP), `sendmsg`(UDP), `recvmsg`(UDP) 시스템 콜이 불릴 때 목적지 IP가 Service IP인지 확인하고, 맞으면 백엔드 하나를 골라 그 자리에서 목적지를 바꿉니다.

W2에서 정리한 kube-proxy 방식은 "kube-proxy가 규칙을 넣고, 커널이 패킷마다 규칙을 거쳐 옮긴다"였습니다. socket-LB는 연결을 여는 순간 한 번 고르고 끝나기 때문에, 그 뒤 패킷에는 Service IP → Pod IP 변환이 필요 없습니다.

```text
socket-LB 없음 (패킷 단위)
Pod → connect(ClusterIP:80) → 패킷 목적지 = ClusterIP
    → 노드의 패킷 단위 로드밸런서가 요청마다 백엔드 Pod로 DNAT → 백엔드 Pod

socket-LB 있음 (소켓 단위)
Pod → connect(ClusterIP:80)
    → cgroup Hook: 목적지가 Service IP인지 확인, 백엔드 하나 선택, 목적지를 백엔드로 교체
    → 이후 패킷 목적지 = 백엔드 Pod → 백엔드 Pod
```

socket-LB에는 부작용이 하나 있습니다. 연결 시점에 백엔드를 이미 골랐기 때문에, 백엔드가 삭제된 뒤에도 Pod가 그 백엔드로 계속 트래픽을 보낼 수 있습니다. 그래서 Cilium agent는 삭제된 백엔드에 연결된 애플리케이션 소켓을 강제로 종료해서 살아 있는 백엔드로 다시 부하분산되게 합니다. 이 기능에 `CONFIG_INET_DIAG`, `CONFIG_INET_UDP_DIAG`, `CONFIG_INET_DIAG_DESTROY` 커널 옵션이 필요합니다.

### 6.3 설치 순서의 닭과 달걀

W3에서 [[Core Kubernetes]]를 읽으며 "CNI 조정 컨트롤러가 API Server에 닿으려면 ClusterIP를 거쳐야 하니 kube-proxy 설치가 선행된다"고 정리했습니다. Cilium 문서는 이 문제를 직접 다룹니다.

kube-proxy 없이 설치하면(`kubeadm init --skip-phases=addon/kube-proxy`) ClusterIP를 만들어줄 주체가 없으므로, Cilium에 `k8sServiceHost`와 `k8sServicePort`로 **API Server 주소를 직접** 알려줘야 합니다. 즉 kube-proxy를 먼저 까는 것 말고도, ClusterIP를 거치지 않고 API Server에 바로 붙는 방법이 있는 것.

### 6.4 부가 모드


| 모드     | 내용                                   |
| ------ | ------------------------------------ |
| DSR    | 백엔드가 클라이언트에게 직접 응답, 출발지 IP 보존         |
| Maglev | 일관된 해싱(consistent hashing)으로 백엔드 선택    |
| Hybrid | TCP는 DSR, UDP는 SNAT                   |
| XDP 가속 | NodePort/LoadBalancer 요청을 드라이버 계층에서 처리 |


참고자료: [Kubernetes Without kube-proxy](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/), [Cilium 성능 튜닝](https://docs.cilium.io/en/stable/operations/performance/tuning/)



## 7. NetworkPolicy / Identity

### 7.1 IP 대신 Identity

문서가 드는 예시: IP 기반 보안이라면 `role=frontend`나 `role=backend` Pod가 하나 뜨거나 내려갈 때마다 **모든 노드의 규칙을 갱신**해야 합니다. Identity 기반이면 새 `role=frontend` Pod는 key-value 저장소에서 자기 Identity만 확인하면 되고, `role=backend` Pod가 있는 노드에서는 아무것도 할 필요가 없습니다.

W2에서 "Pod IP는 바뀔 수 있으니 Service가 고정된 접근 지점을 준다"고 정리했는데, 정책에서 같은 문제를 푸는 방식이 Identity입니다. 바뀌는 IP 대신 잘 안 바뀌는 레이블을 기준으로 삼는 것.

- Identity는 보안 관련 레이블로 정해지고, 클러스터 전체에서 고유한 식별자를 받음
- 같은 보안 레이블의 Endpoint들은 Identity 하나를 공유함 → 애플리케이션 규모가 커져도 많은 Endpoint에 정책을 한꺼번에 적용할 수 있음

Cilium이 관리하지 않는 대상은 예약된 Identity로 표현합니다.


| Identity                  | ID | 대상                 |
| ------------------------- | -- | ------------------ |
| `reserved:unknown`        | 0  | Identity를 못 구함     |
| `reserved:host`           | 1  | 로컬 호스트             |
| `reserved:world`          | 2  | 클러스터 밖             |
| `reserved:unmanaged`      | 3  | Cilium이 관리하지 않는 워크로드 |
| `reserved:health`         | 4  | Cilium health 체크   |
| `reserved:init`           | 5  | 아직 Identity가 정해지지 않음 |
| `reserved:remote-node`    | 6  | 다른 노드              |
| `reserved:kube-apiserver` | 7  | kube-apiserver 백엔드 |
| `reserved:ingress`        | 8  | Ingress 프록시 출발지 IP  |


참고자료: [Identity 기반 보안](https://docs.cilium.io/en/stable/security/network/identity/), [Cilium 용어](https://docs.cilium.io/en/stable/gettingstarted/terminology/)

### 7.2 정책 리소스와 적용 모드

Cilium은 정책 리소스 세 가지를 받습니다: Kubernetes `NetworkPolicy`, `CiliumNetworkPolicy`, 노드 셀렉터까지 쓸 수 있는 클러스터 단위 `CiliumClusterwideNetworkPolicy`.

적용 모드는 세 가지입니다.


| 모드      | 동작                                               |
| ------- | ------------------------------------------------ |
| default | 정책이 Endpoint를 선택하기 전까지는 전부 허용. 선택되면 그 방향만 기본 거부  |
| always  | 정책이 없어도 모든 Endpoint에 적용                          |
| never   | 정책이 있어도 적용 안 함                                   |


default 모드는 W4에서 본 Kubernetes 기본값(정책 없으면 비격리)과 같은 모양입니다. 방향별(ingress/egress)로 따로 기본 거부로 바뀐다는 점도 같습니다.

참고자료: [Cilium 정책 개요](https://docs.cilium.io/en/stable/security/policy/intro/)

### 7.3 L3 / L4 / L7

L3에서 상대를 고르는 방법이 Kubernetes NetworkPolicy보다 넓습니다.


| 방식           | 내용                                          |
| ------------ | ------------------------------------------- |
| Endpoints 기반 | 양쪽이 Cilium이 관리하는 Endpoint일 때 레이블로 지정         |
| Services 기반  | Kubernetes Service 이름이나 레이블 셀렉터로 지정          |
| Entities 기반  | `host`, `world`, `cluster`, `kube-apiserver` 등 미리 정의된 범주 |
| Node 기반      | 특정 노드를 허용하거나 막음                             |
| IP/CIDR 기반   | Endpoint가 아닌 외부 대상, IP나 서브넷을 직접 적음          |
| DNS 기반       | DNS 이름을 조회한 IP로 외부 대상 선택                    |


L7 정책은 노드 로컬 Envoy(DaemonSet이나 agent Pod 안에 내장)를 거쳐 적용됩니다. 문서 예시는 `env=prod` 레이블 Endpoint에서 오는 `GET /public`만 허용하고 나머지 URL이나 메서드는 거부하는 규칙.

L7 위반은 **패킷 drop이 아니라** 프로토콜에 맞는 거부 응답으로 돌아옵니다. HTTP면 `403 Access denied`, DNS면 `REFUSED`. L3/L4 정책 위반이 조용히 drop되는 것과 다르게 보입니다.

W4에서 Calico는 L5~7 조건을 Istio와 함께 쓸 때만 지원한다고 정리했습니다. W7에 파킹해둔 "Calico도 L7 정책이 되는지, Cilium만의 것인지"에 대한 문서 기준 답은: Cilium은 자체 Envoy로 L7 정책을 적용하고, Calico는 Istio와 결합해야 한다는 것입니다.


| 구분       | Kubernetes NetworkPolicy | Calico                 | Cilium                    |
| -------- | ------------------------ | ---------------------- | ------------------------- |
| 범위       | L3/L4                    | L3/L4, L5~7은 Istio와 함께 | L3/L4 + L7 (HTTP, DNS)    |
| 클러스터 단위  | X                        | `GlobalNetworkPolicy`  | `CiliumClusterwideNetworkPolicy` |


참고자료: [L3 정책](https://docs.cilium.io/en/stable/security/policy/layer3/), [L7 정책](https://docs.cilium.io/en/stable/security/policy/layer7/)



## 8. Hubble Overview

Hubble은 Cilium 위에서 서비스 간 통신과 네트워크 인프라를 들여다보는 관측 도구입니다. 문서가 꼽는 기능은 L3/L4, 나아가 L7까지 **서비스 의존성 그래프를 자동으로 찾고**, flow를 서비스 맵으로 시각화하고 필터링하는 것.

구조는 4.1절 표 그대로입니다. 노드마다 Hubble server가 flow를 모으고, Relay가 클러스터 전체를 묶고, CLI와 UI가 Relay에 붙습니다.

문서에 나온 기본 명령:

```text
cilium hubble enable        → Hubble 설정 적용, Cilium Pod 재시작, Relay 배포
cilium hubble port-forward  → 로컬에서 Relay(4245)로 접근
hubble status -P            → Healthcheck 결과, flow 통계, 연결된 노드
hubble observe -P           → flow 조회. 결과에 FORWARDED / DROPPED 등으로 표시
```

W4에서 디버깅할 때 Flannel은 포트와 MTU, Calico는 BGP 피어 상태부터 본다고 정리했습니다. Cilium에서는 `hubble observe`로 **어떤 flow가 전달되고 어떤 flow가 버려졌는지**를 직접 볼 수 있습니다. W4 질문 2("정책이 실제로 적용되고 있는지 어떻게 확인하나")에 Cilium이 내놓는 답이 이 도구입니다.

참고자료: [Hubble 개요](https://docs.cilium.io/en/stable/observability/hubble/index.html), [Hubble 설정](https://docs.cilium.io/en/stable/observability/hubble/setup/)



## 9. 실습

kube-proxy 대체 문서에 실린 Hubble 출력 예시로 6.2절의 소켓 단위 변환이 실제로 어떻게 보이는지 확인해봤습니다. `mediabot` Pod가 `nginx-service`로 요청을 보낸 경우입니다.

```text
default/mediabot (ID:5618) <> default/nginx-service:80 (world)  pre-xlate-fwd   TRACED     (TCP)
default/mediabot (ID:5618) <> default/nginx:80 (ID:35772)       post-xlate-fwd  TRANSLATED (TCP)
default/nginx:80 (ID:35772) <> default/mediabot (ID:5618)       pre-xlate-rev   TRACED     (TCP)
default/nginx-service:80 (world) <> default/mediabot (ID:5618)  post-xlate-rev  TRANSLATED (TCP)
```

- `pre-xlate-fwd`: 변환 전. 목적지가 Service(`nginx-service:80`)
- `post-xlate-fwd`: 변환 후. 목적지가 백엔드 Pod(`nginx:80`)로 바뀜
- `-rev` 두 줄: 응답 방향에서 다시 Service 주소로 되돌림
- 각 Endpoint 옆의 `ID:5618`, `ID:35772`가 7.1절의 Identity

이 추적은 agent가 Pod의 cgroup 경로를 찾아야 동작하고, 안 되면 `cilium-dbg monitor`로 대신 볼 수 있다고 문서에 나와 있습니다.

참고자료: [kind 설정](https://kind.sigs.k8s.io/docs/user/configuration/), [Kubernetes Without kube-proxy](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)



## 10. 질문

1. 새 Pod가 뜬 직후 Identity가 아직 정해지지 않은 동안(`reserved:init`)의 트래픽은 정책상 어떻게 처리되는지?
    - 찾아본 것: L3 정책 문서의 `init` entity 설명에 이 상태는 "보통 Kubernetes가 아닌 환경에서만 관찰된다"고 나와 있습니다. 그렇다면 Kubernetes에서는 왜 거의 안 보이는지가 다음 질문입니다. 문서가 가리키는 Endpoint Lifecycle 절을 더 읽어볼 예정.
2. L7 정책 위반이 drop이 아니라 403으로 돌아온다면, 애플리케이션 입장에서 정책 차단과 애플리케이션 자체의 권한 오류를 어떻게 구분하는지? Hubble에서는 어떻게 다르게 보이는지?
3. socket-LB는 `connect()` 시점에 백엔드를 고르는데, 이미 연결된 상태에서 그 백엔드 Pod가 사라지면 기존 연결은 어떻게 되는지?
    - 찾아본 것: 문서상 Cilium agent가 삭제된 백엔드에 연결된 애플리케이션 소켓을 강제로 끊어서, 애플리케이션이 살아 있는 백엔드로 다시 부하분산되게 합니다.
    - 한계도 적혀 있습니다. 소켓을 끊기 전에 그 소켓이 정말 socket-LB를 거쳤는지 확인용 Map(`cilium_lb4_reverse_sk`)에서 찾는데, 이 Map이 차면 LRU로 밀려난 연결은 확인에 실패해서 종료 대상에서 빠집니다. `connect()` 없이 보내는 UDP 소켓처럼 정리가 제대로 안 되는 경우가 있어서 Map이 점점 차오를 수 있고, 그러면 노드를 재시작하기 전까지 소켓 종료가 불안정해질 수 있다고 합니다.
4. eBPF Map 기본 한도(Service LB 노드당 64k, Policy Endpoint당 16k)를 넘으면 어떤 증상으로 나타나는지?
    - 찾아본 것: 한도를 넘으면 항목이 더 들어가지 않습니다. Service LB Map이 차면 Cilium이 Service 변경을 반영하지 못해서 Service IP 연결이나 새 Service 생성에 영향이 갈 수 있다고 합니다. 64k가 Service 개수가 아니라 항목 수라는 점은 3.3절에 정리했습니다.

참고자료: [L3 정책 (entities)](https://docs.cilium.io/en/stable/security/policy/layer3/), [Kubernetes Without kube-proxy (Limitations)](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/), [Cilium eBPF Maps](https://docs.cilium.io/en/stable/network/ebpf/maps/)