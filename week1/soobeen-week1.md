# Week 1. Kubernetes & Linux Networking Overview

## 1. Kubernetes 컴포넌트 Overview

| 컴포넌트 | 네트워킹 관점에서 하는 일 |
|---|---|
| **API Server** | Service, Endpoint, NetworkPolicy 등 네트워킹 오브젝트 생성 요청이 들어오는 곳 |
| **etcd** | 어떤 Pod가 어떤 IP를 할당받았는지 등 상태 저장소 |
| **Scheduler** | Pod의 Node 배치를 결정 → 배치된 Node의 Pod CIDR 대역에서 IP 할당이 시작됨 |
| **kubelet** | Pod 생성 시 **CNI 플러그인을 직접 호출**하는 주체 (네트워크 셋업 위임) |
| **kube-proxy** | Service의 ClusterIP/NodePort → 실제 Pod로 연결하는 iptables(or IPVS) 규칙을 노드마다 관리 |
| **Container Runtime** | kubelet 지시로 컨테이너 실행 + **Pod의 Network Namespace 최초 생성** (이후 CNI가 채워넣음) |
| **DNS(CoreDNS, 애드온)** | Service 이름 → ClusterIP 변환, Pod의 `/etc/resolv.conf`가 이걸 가리키도록 설정됨 |

> 📌 **중요한 점**
> - **kubelet**: Pod 생성 시 네트워크 셋업을 CNI에 위임
> - **kube-proxy**: 생성된 Pod들 앞에 Service라는 안정적인 주소 부여
> - 즉 **CNI(Pod 개별 네트워크) + kube-proxy(Service 트래픽 분산)**, 이 두 축이 스터디 전체를 관통하는 뼈대

## 2. Linux Network Namespace / veth / bridge

### Network Namespace
- 리눅스 프로세스는 기본적으로 네트워크 스택(인터페이스, 라우팅 테이블, ARP 테이블, iptables 규칙 등) 전체를 공유
- **Network Namespace**: 이 스택을 통째로 복제해 네임스페이스마다 독립된 가상 네트워크 환경을 제공하는 커널 기능
- 새 네임스페이스 생성 시 루프백(`lo`)만 있는 빈 상태로 시작
- **Pod = Network Namespace 1개**
  - 같은 Pod 안 컨테이너들은 이 네임스페이스를 공유 → `localhost`로 서로 통신 가능
- 격리 대상: 인터페이스 목록, IP, 라우팅 테이블, ARP 캐시, iptables 규칙
  - 호스트에서 `ip addr` 실행해도 Pod 내부 인터페이스는 안 보임 (반대도 마찬가지)

### veth pair
- 빈 네임스페이스는 아무 데도 연결되지 않은 고립 상태
- **veth pair**(virtual ethernet pair): 케이블 양 끝을 서로 다른 네임스페이스에 꽂는 가상 이더넷 쌍 → 한쪽으로 들어간 패킷이 그대로 반대쪽으로 나옴
  - 한쪽 끝: Pod 네임스페이스 내부(`eth0`)
  - 반대쪽 끝: 노드의 루트 네임스페이스
- 제약: veth pair 1개는 딱 두 네임스페이스(Pod ↔ 노드)만 연결 가능

### Linux Bridge (가상 스위치)
- Pod가 여러 개면 veth pair로 일일이 연결하는 건 비효율적
- **Linux Bridge**(예: `cni0`)를 노드에 세우고, 각 Pod의 veth 반대쪽 끝을 전부 연결
- 브리지에 물린 인터페이스끼리는 **MAC 주소 기반 자동 스위칭**으로 통신 가능

```
   Pod A netns              node root netns                Pod B netns
 ┌────────────┐                                             ┌────────────┐
 │ eth0(veth) │──veth pair──[ veth-a ]                       │ eth0(veth) │
 └────────────┘                  │                           └────────────┘
                                  │                                 │
                             ┌────┴─────┐                    [ veth-b ]
                             │  cni0    │──────veth pair────────────┘
                             │ (bridge) │
                             └──────────┘
```

> **Kubernetes 연결**
> - 1번 표의 "kubelet이 CNI 호출" = 실제로는 **veth pair 생성 + bridge 연결** 작업
> - CNI마다 연결 방식만 다름 (브리지 대신 라우팅으로 직접 연결하는 Calico 등)
> - 원리는 항상 동일: **네임스페이스를 뭔가로 바깥과 연결**

## 3. TCP/IP Stack

| 계층 | 담당 | Pod 네트워킹에서의 역할 |
|---|---|---|
| L2 (Data Link) | 이더넷, MAC 주소 | veth·bridge가 프레임을 주고받는 계층. 같은 브리지에 물린 인터페이스끼리는 MAC으로만 통신(스위칭) |
| L3 (Network) | IP, 라우팅 | Pod IP 할당, 서로 다른 노드/서브넷 간 경로 결정 |
| L4 (Transport) | TCP, UDP, 포트 | Service의 ClusterIP:Port, kube-proxy의 DNAT 개입 지점 |

- 각 계층은 독립 동작하는 게 아니라, **패킷 하나가 계층을 순서대로 통과하며 계층별 헤더를 씌움(캡슐화)/벗김(역캡슐화)**

```
보낼 때 (캡슐화)                       받을 때 (역캡슐화)
[Application Data]                    [Eth | IP | TCP | Data]
  ↓ L4: TCP 헤더(포트) 부착              ↓ L2: 내 MAC인지 확인
[TCP | Data]                          ↓ L3: 내 IP인지 확인 (아니면 라우팅)
  ↓ L3: IP 헤더(출발/도착 IP) 부착        ↓ L4: 포트 보고 해당 프로세스로 전달
[IP | TCP | Data]                     [Application Data] → 프로세스 도착
  ↓ L2: 이더넷 헤더(MAC) 부착
[Eth | IP | TCP | Data]
```

### 같은 노드 Pod A → Pod B
1. Pod A 애플리케이션이 소켓에 데이터 씀 → 커널이 TCP/IP 헤더 부착
2. Pod A netns의 `eth0`(veth) → 반대쪽 `veth-a`가 브리지(`cni0`)에 등장
3. 브리지가 **L2에서 목적지 MAC 확인** 후 Pod B 쪽 veth로 프레임 전달 (같은 세그먼트라 라우팅 불필요 → 스위칭)
4. Pod B netns에서 역캡슐화 → 애플리케이션 수신

### 다른 노드 Pod A → Pod C (L3 라우팅 개입)
1. Pod A → 브리지까지는 동일
2. 브리지/노드가 목적지 IP(Pod C)의 **자기 서브넷 밖인지** → 라우팅 테이블 조회
3. 물리 NIC로 전달 (CNI가 오버레이 방식이면 VXLAN 등으로 재캡슐화 후 UDP로 전송)
4. 상대 노드 수신 → 역캡슐화 → 자기 브리지 → Pod C의 veth로 전달

## 4. Routing / ARP

### Routing Table
- 커널이 패킷 전송 시마다 조회하는 "목적지별 경로" 표, `ip route`로 확인 가능
- 특정 대역 라우트: 해당 대역은 지정 인터페이스로 직접 전송 (같은 L2, 라우터 불필요)
- default: 위에 안 걸리는 나머지 트래픽 처리

**Pod 네트워킹 적용**
- 같은 노드 Pod ↔ Pod: 로컬 브리지 대역 안 → 라우팅 테이블 조회 없이 L2 스위칭만으로 종료
- 다른 노드 Pod ↔ Pod: 목적지가 로컬 대역 밖 → 라우팅 테이블 조회 → 물리 NIC 또는 터널(오버레이) 인터페이스로 전달
- CNI별로 라우팅 테이블을 채우는 방식이 다름
  - **Calico**: BGP로 노드끼리 서로의 Pod 서브넷 경로를 광고(전파)
  - **Flannel**: 데몬이 각 노드의 라우팅 테이블/오버레이 인터페이스를 직접 설정

### ARP (Address Resolution Protocol)
- IP(논리 주소) → MAC(물리 주소) 매핑을 알아내는 프로토콜
- 브리지 스위칭이 MAC 기준으로 동작 → "이 IP의 MAC이 뭐야?"를 먼저 알아내야 함
1. 브로드캐스트: "이 IP 가진 인터페이스, MAC 알려줘" 전체 전송
2. 해당 IP 소유자만 유니캐스트로 자기 MAC 응답
3. 요청자가 매핑을 ARP 캐시에 저장 (매번 브로드캐스트 방지)

## 5. Netfilter / iptables

- **Netfilter**: 리눅스 커널에 내장된 패킷 필터링/변조 프레임워크
- 패킷이 커널 네트워크 스택을 통과할 때 정해진 지점(훅)마다 정지 → 등록된 규칙에 따라 통과/차단/변조 결정

```
외부 → NIC 진입 → PREROUTING → 라우팅 결정
                                    │
                    ┌───────────────┴───────────────┐
               목적지가 로컬?                    목적지가 다른 곳?
                    │                                │
                 INPUT                            FORWARD
                    │                                │
              로컬 프로세스                        (그대로 통과)
                    │                                │
                 OUTPUT                              │
            (로컬에서 생성된 패킷)                       │
                    │                                │
                    └───────────────┬────────────────┘
                              POSTROUTING
                                    │
                              NIC 송신 → 외부
```

| 훅 | 시점 | Pod 네트워킹에서의 예 |
|---|---|---|
| PREROUTING | 들어오자마자, 라우팅 결정 전 | Service ClusterIP → 실제 Pod IP로 바꾸는 **DNAT** 발생 |
| INPUT | 라우팅 후 목적지가 로컬 프로세스 | 노드 자체를 향한 패킷 |
| FORWARD | 라우팅 후 목적지가 다른 곳(전달) | 브리지를 거쳐 다른 Pod/노드로 넘어가는 트래픽 |
| OUTPUT | 로컬 프로세스가 만든 패킷이 나갈 때 | 노드 자체 프로세스가 보내는 트래픽 |
| POSTROUTING | 실제로 나가기 직전 | Pod → 외부 egress 시 **SNAT/Masquerade** 발생 |

**iptables 테이블** (같은 훅이라도 테이블마다 용도가 다름)

| 테이블 | 용도 | 관련 훅 |
|---|---|---|
| filter (기본) | 허용/차단(방화벽) — NetworkPolicy가 결국 이 규칙으로 구현됨 | INPUT, FORWARD, OUTPUT |
| nat | 주소/포트 변환(SNAT/DNAT) — kube-proxy가 Service 구현에 사용 | PREROUTING, OUTPUT, POSTROUTING |
| mangle | 패킷 헤더 수정(TTL, TOS 등) | 모든 훅 |

- **DNAT**: 목적지 IP/포트 변경. 외부·다른 Pod가 Service ClusterIP로 요청 시 실제 Pod IP로 변환
- **SNAT**: 출발지 IP/포트 변경. Pod가 클러스터 밖으로 나갈 때 노드 IP로 위장

## 정리: 트래픽 흐름 한 번에 이어보기

1. kubelet이 Pod 생성 → 컨테이너 런타임이 **Network Namespace** 생성 → CNI 호출
2. CNI가 **veth pair** 생성으로 Pod netns ↔ 노드 연결, **bridge**에 연결, IP 할당
3. Pod가 보내는 패킷은 **TCP/IP 스택**을 거쳐 캡슐화 → veth → bridge
4. 목적지가 같은 노드면 **L2 스위칭**으로 종료, 다른 노드면 **라우팅 테이블** 조회 후 물리 NIC/터널로 전달
5. Service 경유 통신이면 중간에 **iptables(PREROUTING/POSTROUTING)**의 DNAT/SNAT가 개입해 ClusterIP ↔ 실제 Pod IP 연결
6. 결론: **CNI** = Pod 개별 네트워크 셋업, **kube-proxy/iptables** = Service 트래픽 분산 — 이 두 축이 전체 그림을 구성

## 질문

1. Calico처럼 브리지 없이 라우팅으로만 연결하는 CNI에서는, Pod 입장에서 default 라우트가 무엇으로 잡히는지? (브리지 방식은 게이트웨이가 브리지 IP라 직관적인데 라우팅 전용 방식은 감이 안 잡힘)
2. 같은 노드 내 Pod 간 통신은 브리지에서 L2 스위칭만으로 끝난다고 정리했는데, 이때도 iptables의 FORWARD 체인을 거치는지 아님 브리지 레벨(L2)에서 끝나서 netfilter 자체를 안 거치는지?

## 참고 자료
- [Kubernetes 공식 문서 - Components](https://kubernetes.io/ko/docs/concepts/overview/components/)
- [Kubernetes 공식 문서 - Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
