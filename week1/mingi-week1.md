# Kubernetes & Linux Networking Overview - 1 week

## Kubernetes란?

**컨테이너화된 애플리케이션의 배포, 확장, 관리를 자동으로 처리해주는 오픈소스 Container Ochestration 플랫폼**

## Kubernetes Components

<img width="1415" height="685" alt="image" src="https://github.com/user-attachments/assets/25fcc96b-92ff-48c7-893b-b7c433d9412e" />

### Control Plane Components

마스터 노드 / 클러스터의 전체 상태 관리, 이벤트 감지&대응

- **kube-apiserver**
    - 쿠버네티스 HTTP API를 노출하는 핵심 서버 컴포넌트 (Frontend)
    - **모든 컴포넌트는 apiserver를 통해서만 소통**
    - etcd에 접근할 수 있는 유일한 컴포넌트
    
    | 역할 | 설명 |
    | --- | --- |
    | 인증 | 사용자 및 서비스 계정 인증 |
    | 유효성 검사 | 요청된 리소스의 포맷 및 내용 확인 |
    | 상태 저장 | etcd에 리소스 등록 및 조회 |
    | 통신 중계 | kubelet, controller, scheduler와 연결 허브 역할 |
- **etcd**
    - 특정 시점에 시스템 상태에 대한 일관된 단일 데이터 소스를 제공하는 분산 key-value 데이터 저장소
    - 모든 클러스터 데이터 보유 (pod, service, nodes, etc..)
    - Raft 합의 알고리즘 사용
        
- **kube-scheduler**
    - 아직 노드에 할당되지 않은 파드를 찾아 적절한 노드에 할당하는 컴포넌트
- **kube-control-manager**
    - Controller(reconcile loop)의 집합
    - 여러 Controller가 단일 프로세스 내에서 실행
    - observe → diff → act
- **cloud-controller-manager (선택 사항)**
    - 클라우드별 컨트롤 로직을 포함하는 쿠버네티스 컨트롤 플레인 컴포넌트
    - 클러스터를 클라우드 공급자의 API와 연결
    - 클라우드 환경일 때만 존재

### Node Components

모든 노드 / 실행 중인 파드를 유지하고 쿠버네티스 런타임 환경을 제공

- **kubelet**
    - 클러스터의 각 노드에서 실행되는 에이전트
    - apiserver를 watch 하면서 내 노드에 배정된 파드를 감지
    - PodSpec대로 컨테이너가 잘 돌고있는지 Check
- **container runtime**
    - 컨테이너 실행을 담당하는 소프트웨어
    - kubelet과는 CRI(Container Runtime Interface)라는 표준 인터페이스로 대화
    - CNI 플러그인을 호출하는 주체 (`/etc/cni/net.d` → `/opt/cni/bin`)
- **kube-proxy (선택 사항)**
    - 트래픽을 Service에서 Pod로 라우팅하는 역할
    - Pod ip는 동적으로 변경되는데 이 동적으로 변경하는 ip에 연결을 시도하는건 어려움
    - Pod를 고정된 네트워크 엔드포인트로 만들고, 내부적으로 실제 Pod로 트래픽을 전달 (Service)
    - node의 네트워크 규칙을 관리 (user space, iptables, IPVS)

### Add-ons

클러스터 기능 확장(DeamonSet, Deployment 형태로 배포)

- **대시보드**
    - 웹 대시보드
- **Calico**
    - 네트워킹 및 네트워크 폴리시 제공자
- **Cilium**
    - 네트워킹, 관측 용의성(Observability), 보안 특징을 지닌 eBPF 기반 데이터 플레인을 갖춘 솔루션
- **Flannel**
    - 쿠버네티스와 함께 사용할 수 있는 오버레이 네트워크 제공자
- **OVN-Kubernetes**
    - OVM을 기반으로 하는 쿠버네티스용 오버레이 기반 네트워킹 구현을 제공
- **CoreDNS**
    - 클러스터 내부 DNS

---

## Linux Networking

### namespace

프로세스를 격리된 환경과 자원을 제공하는 가상화 기술

PID, Network, User, etc.. 여러 종류의 namespace가 있다

### Network namespace (Pod)

컨테이너의 네트워크 스택을 격리

Network Interface, ip 주소, 라우팅 테이블, 포트 번호, 방화벽 규칙 등이 격리됨

<img width="932" height="114" alt="image" src="https://github.com/user-attachments/assets/cfcace83-e5b8-4e68-b18b-c6e68f18e08a" />

mingi 네임스페이스 생성

<img width="786" height="78" alt="image" src="https://github.com/user-attachments/assets/1ba47569-d6a6-4873-a2cf-2191e7b80126" />

/var/run/netns 경로에 영속화

<img width="1654" height="520" alt="image" src="https://github.com/user-attachments/assets/f4f69715-154e-4293-8d19-654d6d0dd8ef" />


호스트와 격리됨을 확인 (ip, 라우팅 테이블, ARP 테이블, iptables 규칙 등)

<img width="2284" height="1082" alt="image" src="https://github.com/user-attachments/assets/29cb6e8e-ee11-4b4d-b893-b3ced5da384d" />
호스트 환경

---

### veth (Virtual Ethernet device)

- 리눅스의 Virtual Ethernet Interface
- veth는 항상 쌍으로 만들어짐
- 네트워크 네임스페이스들을 터널로써 연결하거나 물리 디바이스와 다른 네트워크 네임스페이스의 장비를 연결하는 용도

<img width="1620" height="342" alt="image" src="https://github.com/user-attachments/assets/e94f3e5c-ff0e-48d4-9596-147d523f62c1" />

호스트에 veth-a, veth-b 생성

<img width="1632" height="415" alt="image" src="https://github.com/user-attachments/assets/c0f58eb6-c2ec-41ed-85dd-f496bb113a6e" />

veth-a를 mingi 네임스페이스로 이동

호스트:   veth-b@if7   →  "내 짝은 인덱스 7번"
mingi:    veth-a@if6   →  "내 짝은 인덱스 6번"

veth-b는 호스트, veth-a는 네임스페이스 안에 있으나 커널이 인덱스 번호를 전역으로 설정함

- veth up 후 ip 부여 및 ping test

```bash
# 양쪽 UP
sudo ip link set veth-b up
sudo ip netns exec mingi ip link set veth-a up
sudo ip netns exec mingi ip link set lo up

# 네임스페이스 쪽에 IP 부여
sudo ip netns exec mingi ip addr add 10.10.0.2/24 dev veth-a

# 호스트 쪽에도 IP 부여 (지금은 브리지가 없으니 직접)
sudo ip addr add 10.10.0.1/24 dev veth-b

# 통신 확인
ping -c 3 10.10.0.2 # host -> namespace
sudo ip netns exec mingi ping -c 3 10.10.0.1 # namespace -> host
```

<img width="1326" height="595" alt="image" src="https://github.com/user-attachments/assets/db75e739-699f-45fd-a538-080d27f88e41" />

통신이 잘됨을 확인

---

### bridge

- 라우터, 게이트웨이, VM 등에서 패킷을 목적지로 전달(forwarding)하는 역할을 수행
- Bridge를 통해 다른 네트워크 인터페이스와 연결

```bash
# bridge 생성
sudo ip link add br0 type bridge
sudo ip link set br0 up
ip -br link show br0
```

<img width="1644" height="306" alt="image" src="https://github.com/user-attachments/assets/0d1d8852-7ad2-4eea-a1f3-80ed4015ab2d" />

br0 브릿지 생성

```bash
sudo ip link set veth-b master br0 # 호스트의 veth-b를 bridge의 인터페이스로 옮김
bridge link                        # br0에 꽂힌 목록
```

<img width="1716" height="534" alt="image" src="https://github.com/user-attachments/assets/97e88ef9-bfcc-4a7a-a9f4-1874317fc4b1" />

br0에 포트가 꽂혀 UP, LOWER_UP 상태로 전환됨

현 상태

```bash
[mingi]  veth-a(10.10.0.1)  ←veth→  veth-b  ──┐
                                              ├─ br0 (포트 1개)  
[mingi2] (비어있음)                          ──┘
```

- mingi2 네임스페이스안에 veth-c 생성

```bash
sudo ip link add eth0 netns mingi2 type veth peer name veth-c # veth-c 생성
sudo ip link set veth-c master br0                            # veth-c를 브릿지에 붙임
sudo ip link set veth-c up                                    # veth-c 활성화

sudo ip netns exec mingi2 ip addr add 10.10.0.2/24 dev eth0   # eth0에 ip 부여
sudo ip netns exec mingi2 ip link set eth0 up                  
sudo ip netns exec mingi2 ip link set lo up
```

<img width="1918" height="267" alt="image" src="https://github.com/user-attachments/assets/73897071-5d3e-4e08-81c3-8ccefb5c13cb" />

br0에 veth-c가 붙고 mingi2 eth0와 연결된 것을 확인

- 현 상태

```bash
[mingi]  veth-a(10.10.0.1)  ←veth→  veth-b  ──┐
                                              ├─ br0 (포트 2개)  
[mingi2]  eth0(10.10.0.2)   ←veth→  veth-c  ──┘
```

- ping test

<img width="1362" height="601" alt="image" src="https://github.com/user-attachments/assets/57112fdc-685c-4577-9508-24a9813a3fed" />

- mingi 네임스페이스에서 mingi2 (10.10.0.2)로 통신이 되는 것을 확인
- 반대 상황도 잘 되는 것을 확인

---

### Netfilter

- 시스템에 들어오는 모든 패킷을 분석해서 사용자의 뜻대로 처리 및 기록할 수 있도록 하는 프레임워크
- Kernel에 존재하는 Network 관련 Framework로써 원하는 지점에서 Packet 제어를 위한 다섯 가지 Hook(지점)을 제공

```bash
                    ┌─────────────────┐
                    │    라우팅 결정   │
                    └────┬───────┬────┘
                         │       │
  들어옴 → **PREROUTING** ──┤     └──→ **FORWARD** ──→ **POSTROUTING** → 나감
                         │                                   ↑
                         ↓                                   │
                     **INPUT**                           **OUTPUT**
                         ↓                                   ↑
                    로컬 프로세스 ────────────────────────────┘
```

- Chain

| PREROUTING | 인터페이스를 통해 들어온 패킷을 가장 먼저 처리, 목적지 주소의 변경(DNAT) |
| --- | --- |
| INPUT | 인터페이스를 통해 로컬 프로세스로 들어오는 패킷의 처리 (차단/허용) |
| FORWARD | 다른 호스트로 통과시켜 보낼 패킷을 처리, 지나가는 패킷을 처리 (방화벽, IPS) |
| OUTPUT | 해당 프로세스에서 처리한 패킷을 밖으로 내보내는 패킷에 대한 처리 (차단/허용) |
| POSTROUTING | 인터페이스를 통해 나갈 패킷에 대한 처리, 출발지 주소의 변경(SNAT) |
- 주요 기능
    - NAT : 출발지·목적지 주소 변환
    - Packet filtering : 특정 패킷을 차단 또는 허용
    - packet manging : 필요시 패킷 헤더 값을 변경

### iptables

netfilter에 룰을 넣어 사용자가 원하는대로 접근 통제를 할 수 있도록 하는 도구

<img width="1280" height="502" alt="image" src="https://github.com/user-attachments/assets/71e55455-1a52-4688-8604-c036e23177ce" />

| raw (1) | Connection Tracking 시스템을 우회하는 데 사용 (PREROUTING, OUTPUT) |
| --- | --- |
| mangle (2) | 패킷 헤더를 변경하는 데 사용 (모든 체인) |
| nat (3) | 주소 변환 작업(NAT)을 담당  (POSTROUTING, PREROUTING, OUTPUT) |
| filter (4) | 시스템의 패킷 필터링을 담당 (INPUT, FORWARD, OUTPUT) |
| security (5) | 리눅스 보안 모듈인 SELinux에 의해 사용되는 MAC(Mandatory Access Control) 네트워크 관련 규칙 적용 |
0. mingi2 네임스페이스에서 외부로 패킷이 안나가는 것을 확인

<img width="1324" height="80" alt="image" src="https://github.com/user-attachments/assets/96a9c0af-3690-4f38-962d-940e7de59380" />


1. L3 게이트웨이로 전환

```bash
sudo ip addr add 10.10.0.254/24 dev br0 # br0에 ip 부여

ping -c 2 10.10.0.2 # host -> mingi2 namespace ping
```

<img width="1194" height="300" alt="image" src="https://github.com/user-attachments/assets/75eadd99-ef93-420f-9a87-d3182112e287" />

bridge (host) →  mingi2 namespace ping

1. defalut route 추가

```bash
sudo ip netns exec mingi  ip route add default via 10.10.0.254 # mingi route 추가
sudo ip netns exec mingi2 ip route add default via 10.10.0.254 # mingi2 route 추가

sudo ip netns exec mingi2 ip route

# 10.10.0.0/24 서브넷을 넘어 호스트 네임스페이스에 진입하여 호스트(10.0.0.22)와 라우팅
sudo ip netns exec mingi2 ping -c 2 10.0.0.22 
```

1. ip forwarding

```bash
# Linux는 기본으로 자기에게 온 패킷만 처리하고 목적지가 자기가 아닌건 폐기
# 목적지가 노드 자신이 아니더라도 목적지를 보고 패킷을 전달해주는 라우터 역할
 
sysctl net.ipv4.ip_forward

net.ipv4.ip_forward = 1 # 0이면 라우터 역할 X
```

```bash
# ACCEPT로 해놓아야 전달을 허용, DROP시 iptables에서 막힘

sudo iptables -L FORWARD -n | head -1
sudo iptables -P FORWARD ACCEPT # FORWARD chain의 정책을 변경 (DROP -> ACCEPT)
```

1. 외부로 ping

```bash
sudo ip netns exec mingi2 ping -c 3 8.8.8.8
```

<img width="1644" height="237" alt="image" src="https://github.com/user-attachments/assets/947a958e-24ba-4b4e-a18c-f14eb85678c6" />

외부로 나가는 것만 있고 들어오는 것은 없음

10.10.0.2는 임의로 만든 사설 대역으로써 구글이나 중간 라우터가 어디로 응답해야 할지 모른다.

ICMP 패킷에 출발지 주소가 있어도 그 주소로 가는 경로가 없다.

1. MASQUERADE 적용

```bash
# 외부로 나가기 전에(POSTROUTING) 출발지가 우리 파드 대역(10.10.0.0/24) 대역이면 enp3s0 (host interface)로 나갈 때, 출발지를 enp3s0 인터페이스로 바꿔라 (MASQUERADE)
 
sudo iptables -t nat -A POSTROUTING -s 10.10.0.0/24 -o enp3s0 -j MASQUERADE
```
<img width="1522" height="84" alt="image" src="https://github.com/user-attachments/assets/92767cf9-c7c0-4037-875f-56dc350550ec" />

nat table에 POSTROUTING 정책 적용

```bash
mingi2(10.10.0.2) → veth → br0(GW 10.10.0.254) → 라우팅 → FORWARD
  → POSTROUTING/MASQUERADE(src를 10.0.0.22로) → enp3s0 → 8.8.8.8
```

1. 외부 ping test

```bash
sudo ip netns exec mingi2 ping -c 3 8.8.8.8
```

<img width="1654" height="343" alt="image" src="https://github.com/user-attachments/assets/0381d012-5b89-40e0-9a0e-6052401c3de9" />

src가 호스트 ip인 10.0.0.22로 나가는 것을 확인

etc. conntrack

NAT 규칙은 첫 패킷만 적용되고 그 이후 패킷은 conntrack에 기록되어 자동 변환

| 패킷 | 처리 |
| --- | --- |
| 첫 패킷 | nat 테이블 규칙 평가 → conntrack 기록 |
| 이후 모든 패킷 | conntrack 기록대로 자동 변환 (**규칙 안 봄**) |

실습 → 쿠버네티스 대응

| 실습에서 만든 것 | 쿠버네티스에서 | 담당 |
| --- | --- | --- |
| `ip netns add mingi` | 파드의 네트워크 격리 | containerd (pause 컨테이너) |
| `ip link add ... type veth` | 파드와 노드의 연결선 | CNI 플러그인 |
| `br0` | `cni0` / `docker0` | CNI 플러그인 |
| `ip addr add 10.10.0.2/24` | 파드 IP 할당 | CNI IPAM |
| `ip route add default via ...` | 파드의 기본 경로 | CNI 플러그인 |
| `sysctl ip_forward=1` | 노드의 라우터 역할 | kubeadm 사전 요구사항 |
| `iptables MASQUERADE` | 파드 → 외부 SNAT | CNI (`ipMasq: true`) |
| (아직 안 함) DNAT | Service ClusterIP | kube-proxy |

---

### TCP/IP Model

<img width="813" height="613" alt="image" src="https://github.com/user-attachments/assets/549d504b-2961-46c0-a695-d0f2b28fc5a8" />

- **L1 Physical Layer**
    - 실제 물리적 포트
- **L2 Data Link Layer**
    - 홉 단위, MAC 주소를 보고 찾아감
    - **ARP(Address Resolution Protocol)** - IP 주소로 MAC 주소를 알아내는 프로토콜
        - IP만 알고 MAC 주소를 모르면 어느 홉으로 보내야할지 모른다
        - Broadcast로 ARP 요청 / Unicast로 응답
        - MAC 주소 → IP : RARP
- **L3 Network Layer**
    - 종단 간, IP
    - **Routing** - 패킷을 어디로 보낼지 가장 좋은 경로를 선택하는 과정
        - 목적지 IP를 Routing Table과 대조하며 Table에 없을 시 다음 홉으로 넘김
- **L4 Transport Layer**
    - 포트, 어떤 서비스로 보내야할지, TCP/UDP
 
---

- notion 정리 내용
  
  https://fascinated-jobaria-aa1.notion.site/Kubernetes-Linux-Networking-Overview-1-week-3cf833a88b7e803b8650e3fc19df1cab?source=copy_link

- 참고자료
    
    https://kubernetes.io/ko/docs/concepts/overview/components/
    
    https://tech.kakao.com/posts/484
    
    https://www.44bits.io/posts/container-network-2-ip-command-and-network-namespace/ - ⭐
    
    https://jokaknabi.tistory.com/8
    
    https://wisdom-cloud.tistory.com/188
    
    https://jaykos96.tistory.com/31
    
    https://kylo8.tistory.com/entry/Nework-TCPIP-%EB%AA%A8%EB%8D%B8-4%EA%B3%84%EC%B8%B5-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0-Internet-Protocol-Stack#google_vignette
