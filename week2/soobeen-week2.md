# Week 2. Kubernetes Networking & CNI Fundamentals

## 1. Kubernetes Networking Model

- Kubernetes 네트워킹의 3대 기본 원칙
  - 모든 Pod는 NAT 없이 서로 통신 가능 (Pod IP가 곧 실제 통신 주소)
  - 모든 Node는 NAT 없이 모든 Pod와 통신 가능
  - Pod가 보는 자기 IP = 다른 Pod가 보는 그 Pod의 IP (동일)
- Docker 기본 네트워킹(bridge, NAT + 포트 매핑 필요)과 달리, Kubernetes는 처음부터 **"평평한(flat) 네트워크"**를 전제로 설계됨 → Service 없이도 Pod IP만 알면 직접 접근 가능한 이유
- 네트워크 모델 두 축
  - **커널 네트워크**: Node OS가 패킷을 처리하는 기능 → **kube-proxy**가 여기에 Service 전달 규칙을 설정
  - **Pod 네트워크**: Pod들이 각자 IP를 받아 서로 통신하는 가상 네트워크 → **CNI**가 이걸 만드는 도구
- 이 Pod 네트워크가 1주차에서 했던 Network Namespace + veth + bridge/routing!

## 2. kubelet / CRI / CNI 동작 흐름

Pod 하나가 뜰 때 세 컴포넌트가 순서대로 개입함

| 순서 | 주체 | 하는 일 |
|---|---|---|
| 1 | **kubelet** | API Server를 **watch**하다가 자기 Node로 배치된(`nodeName` 확정) Pod를 스스로 발견 → Container Runtime에게 "이 Pod 실행해" 요청 (CRI 통해) |
| 2 | **CRI**(Container Runtime Interface) | kubelet과 컨테이너 런타임(containerd 등) 사이의 표준 인터페이스. `RunPodSandbox` 호출 시 런타임이 **pause 컨테이너 생성 + Network Namespace 생성** |
| 3 | **CNI** | 런타임이 만든 빈 Network Namespace를 넘겨받아 **veth 생성, bridge/라우팅 연결, IP 할당(IPAM), 라우팅 테이블 설정**까지 전부 처리 (`ADD` 커맨드) |
| 4 | kubelet | CNI가 반환한 IP·라우팅 정보를 Pod 상태에 반영 → `kubectl get pod -o wide`에 IP로 표시됨 |

> 📌 **책임 경계 명확히 분리**
> - CRI = "컨테이너를 실행하고 격리 공간(네임스페이스)을 만드는 것"까지만 책임
> - CNI = "그 격리 공간을 네트워크적으로 채우는 것"만 책임
> - kubelet은 **CRI만 호출**함 CNI 설정을 읽고 `/opt/cni/bin/<plugin>` 바이너리를 실제로 exec하는 주체는 kubelet이 아니라 **CRI 런타임**임

**CNI가 DaemonSet + hostNetwork + privileged로 배포되는 이유**
- CNI가 실제로 하는 일: `Pod IP 할당 → Pod 간 route 생성 → iptables/eBPF 네트워크 규칙 설정`
- 이 작업들은 전부 **Node 자체의 네트워크 스택(인터페이스, 라우팅 테이블, iptables/eBPF)을 직접 건드려야** 함
- 그래서 CNI 컴포넌트(Calico Node, Cilium Agent 등)는 거의 예외 없이
  - **DaemonSet**: 모든 Node에 하나씩 떠서 각 Node의 네트워크를 담당
  - **hostNetwork**: Node의 네트워크 네임스페이스를 그대로 사용(iptables, ip route, ip link 같은 명령어는 실행하는 프로세스가 속한 Network Namespace에만 영향을 줌. CNI 에이전트가 격리된 netns에서 이 명령어들을 실행하면, 그 네트워크 스택만 바꾸는 것 -> 다른 Pod나 Node 전체 영향 X -> Node의 루트 네임스페이스 합류 필요)
  - **privileged**: 네트워크 관련 커널 기능(NET_ADMIN 등)을 쓸 권한 부여

## 3. Pod Network / Pod CIDR / IPAM

- **Cluster CIDR**: 클러스터 전체 Pod가 쓸 수 있는 전체 IP 대역 (예: `10.244.0.0/16`)
- **Pod CIDR**: 이 전체 대역을 **Node별로 쪼갠 서브넷** (예: `10.244.1.0/24`는 특정 Node 전용)
  - 실습에서 확인한 Pod IP들(`10.244.1.2`, `10.244.1.3`)이 전부 `10.244.1.x`인 게 우연이 아니라, 같은 Node(`desktop-worker`)의 Pod CIDR 대역에서 나온 것
- **IPAM**(IP Address Management): CNI 안에서 실제로 "이 대역 중 안 쓰는 IP 하나 줘"를 처리하는 서브 플러그인.

- Pod가 삭제되면 IPAM이 그 IP를 회수해서 재사용 대기열에 넣음 — 단, `DEL` 호출이 유실되면(kubelet 비정상 종료 등) IP가 영구히 예약 상태로 남는 문제도 있음 (host-local은 노드 로컬 파일로, Calico는 CRD로 상태를 관리하는 차이 때문에 복구 난이도도 다름)
- **Pod 재생성 시 IP가 바뀌는 이유**: 새 Pod는 IPAM에게 새로 IP를 요청하는 것이라, 같은 Node에 다시 뜨더라도 그 사이 다른 Pod가 이전 IP를 가져갔을 수 있어 보장이 없음 → 실습에서 확인한 "Pod 재생성 → Pod IP 바뀜"이 바로 이 메커니즘

## 4. kube-proxy와 Service

### Service·EndpointSlice·CoreDNS·kube-proxy의 역할 분담

| 컴포넌트 | 왜 필요한가 |
|---|---|
| **Service** | Pod IP가 고정이 안 되므로, Pod 앞에 **안정적인 이름과 가상 IP**를 제공 |
| **EndpointSlice** | Service는 selector로 "대상 조건"만 지정할 뿐, 실제 트래픽을 보낼 **Pod IP 목록**은 별도로 필요 → selector와 Pod label을 비교해서 Ready 상태인 Pod IP만 계속 갱신 |
| **CoreDNS/kube-dns** | 환경(dev/prod)마다 Service의 실제 ClusterIP가 다르므로, 사람이 외우는 "이름"을 실제 통신에 쓰는 "IP"로 변환 |
| **kube-proxy** | 요청이 ClusterIP로 들어왔을 때, 실제로 어떤 Pod IP로 보낼지 **Node 커널에 전달 규칙(iptables/IPVS)을 구성** |

### Service → EndpointSlice 연결
```
Service: web (selector: app=web)
   ↓
EndpointSlice
   - app=web 라벨의 Ready Pod IP 목록 등록
   ↓
kube-proxy가 이 목록을 감시 → ClusterIP(10.96.210.219) 요청을 backend Pod로 전달하는 규칙 구성
```
- Pod가 Running이어도 **Ready가 아니면** 일반적으로 EndpointSlice의 트래픽 대상에서 제외됨 (readinessProbe/readinessGate 실패 시 `ready=false`, not-ready 엔드포인트도 목록에 남긴 함)
- Service 자체는 selector 없이도 만들 수 있음 → 이 경우 EndpointSlice를 직접 관리해서 외부 DB/외부 시스템 IP를 Service처럼 연결하는 용도

### 일반 Service vs Headless Service (DNS 응답 차이)

| 구분 | DNS 응답 | 의미 |
|---|---|---|
| 일반 Service (`nginx`) | ClusterIP **하나** 반환 | Service 계층이 backend Pod로 트래픽 분산 (kube-proxy 개입 필요) |
| Headless Service (`clusterIP: None`) | EndpointSlice에 등록된 **각 Pod IP를 전부** 반환 | 클라이언트가 반환받은 Pod IP 중 하나로 **직접 연결** (kube-proxy 미개입) |

- Headless는 ClusterIP 자체가 없으므로 kube-proxy가 만들 DNAT 규칙도 없음 → **CoreDNS 응답만으로 라우팅이 끝나는 구조**
- StatefulSet과 조합하면 Pod별로 안정적인 DNS 이름이 유지됨

### NodePort / LoadBalancer / Ingress (Service를 외부에 노출하는 세 가지 방식)

| 방식 | 주 역할 | 대표 사용 사례 |
|---|---|---|
| **NodePort** | Node IP + 고정 포트로 Service 노출 | 실습, 단순 노출, 자체 LB 연동 |
| **LoadBalancer** | 클라우드/외부 LB가 Service에 외부 IP 할당 | 클라우드 단일 TCP/UDP Service 공개 |
| **Ingress** | HTTP Host/Path/TLS 기반 **L7 라우팅** | 여러 웹/API Service를 하나의 진입점으로 공개 |

> NodePort·LoadBalancer는 "Service를 외부에 노출하는 방식"이고, Ingress는 "어느 Service로 보낼지 결정하는 라우팅 계층"이라 층위가 다름. Ingress Controller 자체도 외부에서 접근하려면 결국 LoadBalancer, NodePort, 또는 `kubectl port-forward`(로컬 테스트 전용) 중 하나가 필요함.

## 5. Service → Pod Packet Flow 추적


```
① dns-client Pod가 "web IP 알려줘" DNS 질의
   → nameserver인 kube-dns Service: 10.96.0.10:53 로 전송

② kube-dns(ClusterIP)가 CoreDNS Pod로 연결
   → CoreDNS의 kubernetes 플러그인이 API Server의 Service 정보 조회
   → web Service의 ClusterIP: 10.96.210.219 반환

③ dns-client가 10.96.210.219:80으로 실제 HTTP 요청 전송
   → 이 패킷이 Node를 지날 때 iptables PREROUTING에서
     kube-proxy가 구성해둔 DNAT 규칙 적용
   → 목적지가 ClusterIP → 실제 Ready Pod IP(예: 10.244.1.2:80)로 변경

④ 변경된 목적지 IP로 라우팅
   → 같은 Node면 bridge에서 L2 스위칭, 다른 Node면 라우팅 테이블 조회
   → CNI가 만들어둔 veth를 통해 최종적으로 Pod netns 진입

⑤ Pod 내부 nginx 컨테이너:80 도착 → 응답 생성 → 역방향으로 동일 경로 되짚어 전달
```

> 이 흐름에서 **CNI는 ④번(Pod IP까지의 실제 경로)까지만** 책임지고, **①~③번(이름 해석 + ClusterIP→Pod IP 변환)은 CoreDNS와 kube-proxy의 몫**!
>
> 참고로 ③번은 **kube-proxy가 iptables 모드일 때**만 해당되고, IPVS 모드면 `PREROUTING`이 아니라 `LOCAL_IN` 훅에서 처리됨

## 결론

**1. Pod IP 할당 과정은?**
kubelet이 CRI로 Container Runtime을 호출해 Network Namespace를 만들고, CRI 런타임이 CNI 플러그인을 exec해서 그 안에 veth를 심고 IPAM으로 IP 하나를 뽑아 할당함. 단 "어느 대역에서 뽑는지"는 CNI 구현마다 다름. Pod가 재생성되면 IPAM에 새로 요청하는 것이라 IP가 바뀔 수 있음.

**2. Service가 백엔드 Pod로 연결되는 방식은?**
Service는 주소만 제공하고 실제 대상 목록은 EndpointSlice가 들고 있음. CoreDNS가 이름→ClusterIP 변환을, kube-proxy가 ClusterIP→실제 Pod IP로 바꾸는 iptables 규칙을 담당. Headless Service는 이 중 kube-proxy 단계를 생략하고 CoreDNS가 Pod IP를 직접 알려줌.

**3. CNI가 네트워크를 구성하는 책임 범위는?**
Pod가 IP를 받고 다른 Pod/Node와 통신 가능한 경로를 만드는 것까지 책임짐. Service의 가상 IP 개념, DNS 이름 해석, ClusterIP→Pod IP 변환은 CNI 영역이 아니라 각각 kube-proxy/CoreDNS의 몫임. 즉!! CNI는 어디까지나 Pod 네트워크만 책임지고, 커널 네트워크의 Service 규칙은 kube-proxy 담당이라고 이해했음.

## 질문

1. Headless Service는 ClusterIP(고정 주소)가 없어서 kube-proxy가 만들 DNAT 규칙 자체가 없는데, 그럼 kube-proxy 입장에서는 Headless Service에 대해 할 게 없을 거 같음. 정말 아무것도 안 하는지?
2. Pod CIDR가 Node별로 고정 할당되어 있다면, Pod가 재생성되면서 스케줄러가 다른 Node로 배치할 경우 Pod IP의 대역 자체도 통째로 바뀌는 셈인데 이게 클라이언트 쪽 캐시(DNS 캐시, 커넥션 풀 등)에 문제를 일으키지는 않는지?

## 참고 자료
- [Kubernetes 공식 문서 - Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Kubernetes 공식 문서 - Service](https://kubernetes.io/ko/docs/concepts/services-networking/service/)
- [CNI 공식 스펙](https://github.com/containernetworking/cni)
