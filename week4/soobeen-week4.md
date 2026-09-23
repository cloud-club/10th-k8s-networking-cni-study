# Week 4. Primary CNI

최근에 네트워크 지식이 부족한 거 같아서 NAT vs Bridge 실습했는데 그와 매핑하는 식으로 작성했습니다!

> **이번 주 훑어보기**
> - Cross-Node 통신 문제 = 물리망이 Pod CIDR을 모른다는 것 하나뿐 → 해법은 캡슐화(오버레이) 아니면 라우팅 등록(언더레이) 둘 중 하나
> - **Flannel**: VXLAN으로 감싸서(오버레이) 전달, 노드 안쪽은 표준 bridge 플러그인에 그대로 위임
> - **Calico**: BGP/IPIP/VXLAN/CrossSubnet 4가지 모드 지원, 브리지 없이 `/32` 라우팅으로 노드 자체를 라우터로 만듦
> - 둘의 차이는 결국 "캡슐화를 어디서 벗기고 붙이는가"와 "브리지를 거치는가"로 좁혀짐 → 후자가 NetworkPolicy가 전 구간에서 먹히는지까지 결정

## 1. Cross-Node 통신의 근본 문제

물리 네트워크(스위치/라우터)는 노드 IP만 알고, Pod CIDR은 모름

```
podA(10.244.0.11) → nodeA(192.168.100.1) → [물리망] → nodeB(192.168.100.2) → podB(10.244.1.11)
```

물리망 입장에서 `10.244.0.0/16` 같은 Pod 대역은 자기 라우팅 테이블에 없는 미지의 주소

해결 방법은 원리적으로 두 가지뿐:

| 접근 | 방법 | 대가 |
|---|---|---|
| 오버레이 (캡슐화) | Pod 패킷을 노드 IP 헤더로 한 번 더 감쌈 → 물리망은 바깥 헤더(노드 IP)만 보고 라우팅 | 오버헤드, MTU 축소, CPU 캡슐화 비용 |
| 언더레이 (라우팅) | 물리망 또는 각 노드의 라우팅 테이블에 Pod CIDR 경로 자체를 등록 | 물리망 협조 필요 (BGP 피어링, 동일 L2 등) |

Flannel = 기본적으로 전자, Calico = 기본적으로 후자 (둘 다 다른 모드로 전환 가능)

> 🔗 **실습 연결**: VMware NAT vs Bridge 실습과 원리가 똑같은 구조
> NAT 모드 = VM의 진짜 IP를 감춘 채 호스트가 대신 통신(오버레이 느낌, 상대는 실체를 모름)
> Bridge 모드 = VM이 자기 정체성 그대로 물리망에 노출(언더레이 느낌, 물리망이 실체를 직접 인식)
> Flannel VXLAN이 NAT처럼 "숨기고 감싸는" 쪽, Calico BGP가 Bridge처럼 "그대로 노출"하는 쪽에 대응

---

## 2. Flannel: VXLAN 캡슐화

### 2.1 VXLAN 프레임 구조

VXLAN = L2 프레임(내부 Ethernet)을 UDP 페이로드에 실어서 L3 위로 나르는 캡슐화 방식

```
[외부 Ethernet 14B] [외부 IP 20B] [UDP 8B] [VXLAN Header 8B] [내부 Ethernet 14B] [내부 IP] [Payload]
                     └──────────────── 오버헤드 50B ────────────────┘
```

오버헤드 = 외부 IP(20B) + UDP(8B) + VXLAN(8B) + 내부 Ethernet(14B) = **50 Byte**
→ 외부 Ethernet 14B는 MTU 계산에 안 넣음 (L2 헤더는 매 홉마다 갈아끼워지므로)

VXLAN 헤더 8B 중 24bit = VNI(Virtual Network Identifier, 논리 네트워크 구분자) → Flannel 기본값 `1`

포트:
- IANA 표준 `4789` → Calico VXLAN이 이걸 씀
- Flannel은 커널이 표준화 이전에 채택한 `8472`를 그대로 유지 → 방화벽 규칙 짤 때 이 차이 놓치면 캡슐화 트래픽 막힘

### 2.2 MTU가 줄어드는 구조

```
Underlay(물리 NIC) MTU 1500
                     │  VXLAN 오버헤드 -50
                     ▼
flannel.1(VXLAN 인터페이스) MTU 1450
```

커널이 VXLAN 디바이스 생성 시 부모 인터페이스 MTU - 50을 자동 설정

```bash
$ cat /sys/class/net/flannel.1/mtu   # → 1450
$ cat /sys/class/net/eth0/mtu        # → 1500
```

프레임 크기로 검증 (`ping -s 1000`):
```
캡처된 프레임 1092B - 외부 Ethernet 14B = 외부 IP 패킷 1078B
내부 IP 패킷 = 1000(payload) + ICMP 8B + IP 20B = 1028B
1078 - 1028 = 50  ← VXLAN 오버헤드와 정확히 일치
```

경계값 실측:
```
$ ping -M do -s 1423   → 100% loss   (1423+28(ICMP+IP) = 1451 > 1450)
$ ping -M do -s 1422   → OK          (1422+28 = 1450, 정확히 한계)
```
`-M do` = DF(Don't Fragment) 비트 켜서 중간 조각 안 나고 그대로 통과하는지 확인
→ MTU 잘못 잡았을 때 조용히 성능만 깎이는 대신 명확하게 실패로 드러나게 하는 방법

### 2.3 flanneld와 CNI 플러그인은 "파일 하나"로 느슨하게 연결

저번주 정리: "노드 간(②)만 직접 구현, 노드 안(①)은 표준 bridge 플러그인에 위임" → 이 위임이 실제로 어떻게 일어나는지?

```
flanneld (DaemonSet)
  ① API 서버에서 이 노드의 podCIDR 확인
  ② /run/flannel/subnet.env 에 기록
       FLANNEL_SUBNET=10.244.1.0/24
       FLANNEL_MTU=1450
  ③ VXLAN 장치(flannel.1) 생성·라우트·FDB 관리 ← Flannel의 진짜 담당 영역
        │
        │ (직접 호출 아님, 파일로만 연결)
        ▼
flannel CNI 플러그인 (실행 파일, ADD 호출 시 실행되고 종료)
  ④ subnet.env 읽음
  ⑤ { "type": "bridge", "bridge": "cni0", "ipam": {"type": "host-local", ...} } 설정을 즉석에서 만들어냄
  ⑥ bridge 플러그인 호출(위임)
        │
        ▼
reference bridge 플러그인이 실제로 veth 만들고 cni0에 연결 (Flannel 고유 코드 아님, 표준 플러그인 재사용)
```

→ "Flannel CNI"의 노드 안쪽 데이터패스는 사실 Flannel이 만든 게 아니라 표준 bridge 플러그인 그대로
→ Flannel 고유 코드는 오직 flanneld가 관리하는 백엔드(VXLAN 터널) 부분뿐
→ 이게 Flannel이 "단순하다"고 평가받는 이유 →  새로 짜야 할 로직 자체가 원래 적음

> 🔗 **실습 연결**: `bridge` 타입 플러그인이 만드는 `cni0`가 Docker 실습에서 계속 봐온 `docker0`/`br-9cc12878a83e`와 완전히 같은 종류(Linux Bridge). veth pair 생성 방식(`vethXXXX@ifN master br-...`)도 동일한 메커니즘. Docker가 컨테이너 만들 때 하던 일을 Flannel이 Pod 만들 때 그대로 하는 것.

### 2.4 VTEP과 세 가지 커널 엔트리

`flannel.1`이 VXLAN VTEP 역할. nodeA → nodeB 패킷 전달 시 커널이 세 번 조회:

| 엔트리 | 답하는 질문 | 예시 값 |
|---|---|---|
| route | 어느 디바이스로? next-hop은? | `10.244.1.0/24 via 10.244.1.0 dev flannel.1 onlink` |
| neigh | next-hop의 MAC은? | `10.244.1.0 lladdr <MAC_B> dev flannel.1 PERMANENT` |
| fdb | 그 MAC은 어느 VTEP(노드 IP)으로 캡슐화? | `<MAC_B> dev flannel.1 dst 192.168.100.2` |

- next-hop `10.244.1.0` = nodeB Pod CIDR의 네트워크 주소 자체 → Flannel이 각 노드의 `flannel.1`에 `/32`로 이 주소를 부여해 "이 주소로 가면 결국 그 노드"라는 규약을 만든 것
- `onlink` 플래그: next-hop이 실제로는 같은 서브넷이 아닌데도 "같은 링크에 있다고 간주하고 보내라"고 커널에 강제 지시
- neigh 엔트리가 `PERMANENT`인 이유: VXLAN 터널 너머의 상대는 실제 이웃이 아니라 가상 이웃이라 ARP 브로드캐스트가 안 통함 → flanneld(컨트롤 플레인)가 API 서버로 다른 노드 정보를 미리 알아내서 정적으로 박아둠 → `nolearning` 옵션과 세트

```bash
$ ip -d link show flannel.1
... vxlan id 1 dev eth0 dstport 8472 nolearning ...
```

`nolearning` = 데이터플레인에서 자동으로 MAC 학습 안 함 → 대신 flanneld가 컨트롤 플레인에서 모든 엔트리를 명시적으로 기록
→ VXLAN 표준 스펙의 멀티캐스트 기반 학습이 아예 불필요 (Kubernetes API라는 기존 중앙 저장소를 그대로 활용)

> 🔗 **실습 연결**: route/neigh/fdb 세 엔트리 개념이 낯설지 않은 이유 → Docker 실습에서 `iptables -t nat -L DOCKER`로 DNAT 규칙, `ip addr`로 veth/브리지 연결을 직접 확인했던 것과 같은 층위. 다만 Docker는 DNAT(주소 변환)으로 해결하고, Flannel VXLAN은 캡슐화(주소를 감싸서 운반)로 해결한다는 게 방식의 차이.

### 2.5 실제 캡슐화 캡처와 TTL로 홉 수 확인

```bash
$ ip netns exec podA ping -c 3 10.244.1.11
64 bytes from 10.244.1.11: icmp_seq=1 ttl=62 time=0.664 ms
```

| 구성 | TTL | 경유 라우터 |
|---|---|---|
| 같은 노드, 브리지형 | 64 | 0: L2 스위칭이라 라우팅 홉이 없음 |
| 같은 노드, 라우팅형 | 63 | 1: 호스트 커널 1회 |
| 크로스 노드(Flannel) | 62 | 2: nodeA, nodeB 둘 다 라우팅 |

TTL이 2 줄어든 것 = VXLAN 캡슐화 자체는 홉 수에 안 잡히고, "Pod 관점에서 nodeA와 nodeB라는 두 개의 라우터를 거쳤다"는 사실만 보임
→ 캡슐화는 물리 계층 아래서 일어나는 일이라 IP 헤더의 TTL 카운트에는 안 나타남


```bash
$ tcpdump -i eth0 -nn -T vxlan 'udp port 8472'
IP 192.168.100.1.46607 > 192.168.100.2.8472: VXLAN, flags [I] (0x08), vni 1
IP 10.244.1.11 > 10.244.0.11: ICMP echo reply, id 2636, seq 1, length 64
```

- 물리 NIC에서 **Pod IP로 필터링하면 아무것도 안 걸림** : 물리 NIC이 보는 건 캡슐화된 바깥 패킷(노드 IP)뿐. `udp port 8472` 조건을 걸거나 `flannel.1` 인터페이스에서 캡처해야 Pod IP가 보임

> 🔗 **실습 연결**: 브리지(`br-9cc12878a83e`)에서 캡처하면 DNAT 후 컨테이너 IP:80이 보이고, 호스트 인터페이스(`ens160`)에서 캡처하면 아직 변환 전 IP:8081만 보였던 것. "어느 지점에서 캡처하느냐에 따라 보이는 주소가 다르다"는 원리가 여기서도 적용

### 2.6 host-gw: 캡슐화 없는 대안

```bash
$ ip route add 10.244.1.0/24 via 192.168.100.2 dev eth0
```

VXLAN 라우트를 걷어내고 상대 노드 IP를 직접 next-hop으로 등록 → 캡슐화 없이 Pod IP가 물리망에 그대로 노출된 채 전달

```bash
$ tcpdump -i eth0 -nn
IP 10.244.0.11 > 10.244.1.11: ICMP echo request   ← 캡슐화 없이 Pod IP가 그대로 보임
```

- MTU 1500 전부 사용 가능 (캡슐화 오버헤드 없으니까)
- 전제조건: **모든 노드가 같은 L2 세그먼트**여야 함 → 서로를 직접 next-hop으로 지정할 수 있어야 하기 때문. 노드가 여러 서브넷에 흩어지면 이 방식 자체가 성립 안 함

**PMTU 캐시 함정**: VXLAN → host-gw로 전환 직후 `ping -M do -s 1472`(1500-28=1472, 이론상 성공해야 함)가 실패하는 경우 있음

```bash
$ ip route get 10.244.1.11
10.244.1.11 via 10.244.0.1 dev eth0 src 10.244.0.11
    cache expires 550sec mtu 1450   ← 예전 VXLAN 시절 캐시된 PMTU가 남아있음
```

`ip route flush cache` 하면 정상 복구
→ 운영 시사점: MTU를 바꾸거나 CNI 모드를 전환해도 기존에 떠 있던 Pod는 최대 10분가량 캐시된 PMTU를 그대로 들고 있음 → 즉시 검증하면 실패로 보이지만 실제로는 설정 문제가 아니라 캐시 문제일 수 있음 → Pod 재시작이나 캐시 flush 필요

---

## 3. Calico: 각 노드를 라우터로

### 3.1 모드별 비교 (BGP / IPIP / VXLAN / CrossSubnet)

| 모드 | 캡슐화 | 디바이스 | 오버헤드/MTU | 비고 |
|---|---|---|---|---|
| BGP (no overlay) | 없음 | `eth0` | 0 / 1500 | 물리망 BGP 피어링 필요 |
| IPIP | IP-in-IP (proto 4) | `tunl0` | 20 / 1480 | IPv4 전용, 일부 클라우드(Azure)가 proto 4 차단 |
| VXLAN | UDP 4789 | `vxlan.calico` | 50 / 1450 | **BGP 불필요** → Felix가 직접 라우트 기록 |
| CrossSubnet | 목적지에 따라 혼재 | `eth0`/`tunl0` | 혼재 | Calico 권장 구성 |

IPIP가 VXLAN보다 오버헤드가 작은 이유는 단순하다 — **IPIP는 L3(IP 패킷)만 한 번 더 감싸는데, VXLAN은 L2(Ethernet 프레임)까지 통째로 감싸기 때문에** 내부 Ethernet 헤더와 UDP 헤더만큼 더 크다.

VXLAN 모드가 BGP 없이도 동작하는 이유: 노드끼리 경로 정보를 실시간 교환할 필요 없이 **Felix가 Kubernetes API(datastore)에서 노드 정보를 읽어서 직접 라우트와 FDB를 프로그래밍**하기 때문. UDP 캡슐화라 어디서나(어떤 방화벽·클라우드든) 통과하기 쉬운 것도 장점.

### 3.2 CrossSubnet: 조건부 캡슐화

목적지 노드가 같은 서브넷 → 직결(캡슐화 없음), 다른 서브넷 → 캡슐화

판정 로직 (`felix/calc/l3_route_resolver.go`):
```go
nowSameSubnet := myNewNodeInfoKnown && myNewV4CIDR.ContainsV4(otherNodesIPv4)
```
= "상대 노드 IP가 내 호스트 CIDR 안에 들어오는가?"

```
10.244.175.0/24 via 172.18.0.4 dev eth0   proto 80 onlink   ← 같은 서브넷: 직결
10.244.99.0/24  via 172.19.0.7 dev tunl0  proto 80 onlink   ← 다른 서브넷: 캡슐화
```

같은 라우팅 테이블 안에 두 형태가 동시에 존재 가능 = CrossSubnet의 특징

⚠️ 노드가 자기 호스트 CIDR을 `/32`로 잘못 보고하면 모든 상대가 "다른 서브넷"으로 판정 → 사실상 Always(항상 캡슐화)로 퇴화. 클라우드 환경에서 노드 인터페이스 prefix 길이 오인식 시 흔히 발생.

### 3.3 BGP 토폴로지 3종

| 토폴로지 | 구조 | 확장성 |
|---|---|---|
| Full mesh (기본) | 모든 노드 ↔ 모든 노드 | N(N-1)/2 세션, ~50 노드까지 |
| Route Reflector | 노드는 RR하고만 피어링 | N 세션, 수백 노드 |
| ToR 피어링 | 노드 ↔ 랙 물리 스위치 | DC 전역 라우팅 |

Full mesh는 노드 수 늘수록 세션 수가 제곱으로 증가 → 확장성 한계 명확
Route Reflector = "모든 노드가 RR 하나만 보고, RR이 대신 뿌려준다" → 세션 수 N개로 감소

**ToR 피어링** = 가장 강력한 구성. 노드가 랙의 물리 스위치와 직접 BGP로 붙어서 **Pod IP가 데이터센터 전체에서 라우팅 가능**해짐 → 외부 레거시 시스템이 로드밸런서 없이 Pod IP로 직접 접근 가능
→ 대가: 물리망과의 강한 결합, 네트워크 팀 협조 필요, Pod CIDR이 물리망 주소 계획에 종속

---

## 4. Pod-to-Pod 패킷 흐름 비교

### Flannel VXLAN
```
1. Pod에서 패킷 생성 (10.244.1.96 → 10.244.2.10)
2. eth0 → veth0 → cni0(Linux bridge)를 거쳐 호스트 라우팅 경로로 진입
3. flannel.1에서 VXLAN 캡슐화 (외부 IP: 노드A IP → 노드B IP)
4. 노드 간 UDP/VXLAN 패킷으로 전달
5. 목적지 노드에서 역캡슐화 후 목적지 Pod로 전달
```

### Calico Direct (BGP, 비캡슐화)
```
0. (사전조건) BIRD가 BGP로 원격 Pod 경로 학습 → 커널 라우팅 테이블에 반영
   Felix가 로컬 Pod 경로(veth)와 정책(iptables) 설정
1. Pod에서 패킷 생성 (10.244.1.10 → 10.244.2.20)
2. Pod A의 eth0 → veth 쌍 → 호스트 인터페이스 caliA로 전달
3. Node 1 커널이 정책 검사 후 라우팅 테이블에서 Node 2를 next-hop으로 선택
4. 캡슐화 없이 원본 Pod IP 그대로 노드 간 전달 (터널 헤더 추가 없음)
5. Node 2에서 caliB → veth → Pod B eth0로 전달
```

Flannel과 결정적 차이: **bridge(cni0)가 없음**. Pod의 veth가 브리지를 거치지 않고 호스트에 직접 붙어서, 호스트가 그 Pod를 `/32` 호스트 라우트로 자기 라우팅 테이블에 등록 → "브리지 경유 L2 스위칭"이 아니라 처음부터 끝까지 순수 L3 라우팅

### Calico IP-in-IP
```
1~4. Direct와 동일 (BIRD/Felix가 경로·정책 설정, Pod 패킷 생성, caliA로 전달, 라우팅 결정)
5. IP-in-IP 캡슐화: 내부 Pod IP를 유지한 채 외부 IP 헤더(노드A IP → 노드B IP) 추가
   중간 라우터는 외부(노드) IP 기준으로만 전달
6. Node 2에서 tunl0가 역캡슐화, 외부 헤더 제거하고 내부 목적지(Pod IP) 확인
7. 목적지 Pod로 전달
```

세 방식의 차이는 결국 **"어디서 캡슐화를 벗기고 붙이느냐"**와 **"브리지를 거치느냐"** 두 가지로 좁혀짐

| | Flannel VXLAN | Calico Direct(BGP) | Calico IP-in-IP |
|---|---|---|---|
| 브리지(cni0) 경유 | O | X | X |
| 캡슐화 지점 | flannel.1 (VXLAN) | 없음 | tunl0 (IP-in-IP) |
| 노드 간 노출 정보 | 노드 IP만 (Pod IP는 숨김) | Pod IP 그대로 노출 | 노드 IP만 (Pod IP는 숨김) |
| 로컬 데이터패스 | L2 스위칭(브리지) | `/32` L3 라우팅 | `/32` L3 라우팅 |

---

## 5. Overlay와 Underlay

핵심은 하나다: **오버레이/언더레이는 절대적 구분이 아니라 상대적 구분**이다. 인터넷망도 VPN 입장에서 보면 언더레이가 되는 것처럼, Flannel VXLAN 관점에서 물리망이 언더레이지만 그 물리망 자체도 더 아래 계층(예: DC 백본 MPLS)에서 보면 누군가의 오버레이일 수 있다.

- Flannel VXLAN = Pod Network가 오버레이, 물리망이 언더레이인 전형적 구조
- Calico Native(BGP) = 오버레이 계층 자체가 없이 Pod Network가 언더레이 라우팅 테이블에 직접 노출 → "언더레이 방식"

> 🔗 **실습 연결**: VM 안의 Docker 브리지(172.18.x)는 VM 입장에선 오버레이지만, VM 자체가 물리망(공유기)에는 NAT(가상, 안 보임) 또는 Bridge(실제 참여)로 붙는 게 한 단계 더 있었던 것. 계층이 여러 겹 쌓이면 "이게 오버레이냐 언더레이냐"는 항상 "누구 기준으로 보느냐"에 달림.

---

## 6. NetworkPolicy: 스펙 밖의 영역이 실제로 어떻게 채워지는가

| CNI | NetworkPolicy | 확장 |
|---|---|---|
| Flannel 단독 | 미지원, 리소스 생성해도 무반응, 에러도 없음 | — |
| Calico | 지원 | `GlobalNetworkPolicy`, `order`, DNS 정책, 서비스 계정 셀렉터 |
| Cilium | 지원 | `CiliumNetworkPolicy`: L7, Identity 기반 (5주차 예정) |

### 6.1 Felix의 집행 구조: iptables 체인 계층

Felix는 정책을 iptables(또는 nftables) 체인 + ipset으로 번역

```
1. cali-INPUT / cali-FORWARD / cali-OUTPUT       ← 진입점, kube-proxy 체인보다 먼저에 위치
2. cali-from-wl-dispatch / cali-to-wl-dispatch   ← 인터페이스 이름으로 분기
3. cali-fw-caliXXXX / cali-tw-caliXXXX           ← 엔드포인트(Pod)별 체인
4. cali-pi-<policy> / cali-po-<policy>           ← 정책별 체인
5. ipset 매칭 (cali40s:...)                       ← 셀렉터에 매칭된 실제 Pod IP 목록
```

핵심 두 가지
- **인터페이스 이름 = 엔드포인트 정체성**: Pod마다 전용 `caliXXXX` 인터페이스가 있어서 "이 트래픽이 어느 Pod에서 왔는가"를 데이터플레인 레벨에서 위조 불가 (IP 스푸핑 걱정 없는 구조)
- **셀렉터는 결국 ipset**: Pod가 늘거나 줄 때 룰(체인) 자체를 재작성하는 게 아니라 ipset의 원소만 갱신 → 대규모 클러스터에서 Pod 증감이 잦아도 iptables 룰 개수가 폭발하지 않는 이유

> 🔗 **실습 연결**: `iptables -t nat -L DOCKER`로 DNAT 규칙 보고 `iptables -L FORWARD`로 DOCKER-USER/DOCKER-FORWARD 체인이 firewalld보다 먼저 걸리는 걸 확인했던 것과 같은 종류. "여러 체인이 순서대로 jump하며 검사된다"는 원리가 여기 Felix의 cali-* 체인 계층에도 그대로 적용됨

### 6.2 정책 평가 순서: K8s NetworkPolicy vs Calico NetworkPolicy

| | 쿠버네티스 NetworkPolicy | Calico NetworkPolicy |
|---|---|---|
| 액션 | allow만, 다중 정책은 OR로 합산 | allow / deny / log / **pass** |
| 순서 | 없음(모두 동시 평가) | `order`(낮은 값이 우선) |
| 범위 | 네임스페이스 | 네임스페이스 + Global |
| 셀렉터 | 레이블 | 레이블 + 표현식 |

`pass` 액션 = `GlobalNetworkPolicy`에서 조건이 맞으면 최종 판단을 네임스페이스 단위 정책으로 위임
→ "플랫폼 팀이 클러스터 전역 기본 정책 배치 → 그 조건 통과한 트래픽만 앱 팀이 자기 네임스페이스 정책으로 세부 판단"하는 계층 구조 가능
→ K8s 표준 NetworkPolicy엔 이런 위임 개념 자체가 없음

### 6.3 브리지형 CNI의 근본적인 정책 공백

```bash
sysctl net.bridge.bridge-nf-call-iptables   # 1이어야 함
```

브리지 기반 CNI(Flannel 등)에서는 같은 노드 안의 Pod 간 트래픽이 L2 스위칭으로 처리
→ `br_netfilter` 커널 모듈 미적용 시, 이 L2 스위칭 트래픽은 iptables/netfilter를 아예 안 거치고 지나감
→ NetworkPolicy를 구현체가 붙여놨어도 **같은 노드 내부 통신에서는 정책이 전혀 안 먹히는 구조적 공백** 발생 가능

Calico는 애초에 브리지가 없고 항상 L3 라우팅 경로를 타서 이런 공백이 구조적으로 없음
→ "Calico가 NetworkPolicy를 잘 지원한다"는 말의 이면 = 사실 "브리지가 없어서 우회 경로 자체가 없다"는 아키텍처적 이유

---

## 7. 한눈에 정리

| | Flannel VXLAN | Flannel host-gw | Calico IPIP | Calico VXLAN | Calico BGP |
|---|---|---|---|---|---|
| 오버헤드 | 50B | 0 | 20B | 50B | 0 |
| Pod MTU | 1450 | 1500 | 1480 | 1450 | 1500 |
| 물리망 요구 | 없음 | 같은 L2 | 없음 | 없음 | BGP 피어링 |
| 로컬 데이터패스 | 브리지 | 브리지 | `/32` 라우팅 | `/32` 라우팅 | `/32` 라우팅 |
| 컨트롤 플레인 | flanneld | flanneld | BIRD/Felix | Felix only | BIRD |
| NetworkPolicy | ✗ | ✗ | ✓ | ✓ | ✓ |

---

## 질문

1. Calico VXLAN 모드는 BGP 없이 Felix가 직접 라우트를 기록한다고 했는데, 이 모드에서 BIRD 프로세스는 아예 안 뜨는 건지, 아니면 떠 있지만 안 쓰이는 건지? Route Reflector 같은 BGP 인프라를 아예 준비 안 해도 되는 건지

2. `br_netfilter`가 꺼져 있으면 같은 노드 내 NetworkPolicy가 안 먹는다는 걸 확인했는데, 이게 Flannel + Calico Policy(Canal 조합)에서도 똑같이 적용되는 문제인지, Canal은 이 부분을 별도로 보정하는 장치가 있는지

---

## 참고 자료

- [CNI 공식 스펙](https://github.com/containernetworking/cni)
- [Flannel 공식 문서](https://github.com/flannel-io/flannel)
- [Calico 공식 문서](https://docs.tigera.io/calico/latest/about/)