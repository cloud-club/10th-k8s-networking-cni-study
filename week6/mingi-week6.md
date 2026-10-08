# Multi-Networking & Secondary Networking - week 6

## Multi-Networking

**하나의 Pod를 여러 네트워크에 연결하여, 용도별로 다른 인터페이스와 통신 경로를 사용하는 구성**

일반적인 Pod는 `lo`를 제외하면 기본 네트워크 인터페이스인 `eth0`를 사용한다. Multi-Networking에서는 여기에 `net1`, `net2` 등의 인터페이스를 추가한다.

```
ㅋPod
├─ lo    → Pod 내부 localhost 통신
├─ eth0  → Kubernetes 기본 네트워크
├─ net1  → 스토리지 네트워크
└─ net2  → 외부 장비 / 고성능 데이터 네트워크
```

- 사용 목적
    - **트래픽 분리**: 일반 서비스 요청과 대용량 스토리지 트래픽의 경로 분리
    - **기존 네트워크 연결**: Pod를 기존 VLAN이나 외부 장비의 네트워크에 연결
    - **고성능 통신**: SR-IOV VF를 이용하여 호스트의 가상 스위칭 경로를 줄임
    - **네트워크 기능 구현**: 방화벽·라우터 같은 워크로드에 여러 연결 제공

**Multi-NIC Pod라고 해서 인터페이스마다 물리 NIC가 하나씩 필요한 것은 아니다.** 하나의 물리 NIC 위에 macvlan·ipvlan 인터페이스나 여러 VF를 구성할 수도 있다. 물리 링크를 공유하면 그 링크의 대역폭과 장애 영향도 공유한다.

## Multus Architecture

#### Multus란

Multus CNI는 쿠버네티스(Kubernetes) 환경에서 하나의 Pod(포드)에 **여러 개의 네트워크 인터페이스를 연결**할 수 있게 해주는 **CNI 플러그인**이다.

- 특징
    - 기존 CNI 플러그인(Calico, Flannel, Cilium 등)을 대체하는 것이 아니라, 여러 CNI 플러그인을 함께 호출하여 Pod에 보조(Secondary) 네트워크 인터페이스를 추가해 주는 '**멀티플렉서**' 역할
    - 포드가 네트워크 집약적이고 SR-IOV 같은 데이터플레인 가속 기술을 지원하는 추가 네트워크 인터페이스가 필요할 때 유용
    - Multus는 단독으로 배포할 수 없고 Kubernetes 클러스터 네트워크 요구 사항을 충족하는 기존 CNI 플러그인이 하나 이상 필요하다

<img width="800" alt="image" src="https://github.com/user-attachments/assets/2d7a60fa-3cd2-4d11-a0d8-7309fa06e432" />


#### 동작 방식

```python
kubelet
   │ CRI: Pod sandbox 생성 요청
   ▼
Container Runtime
   │ CNI 호출
   ▼
Multus
   ├─ Primary CNI   → eth0 구성
   ├─ macvlan CNI   → net1 구성
   └─ SR-IOV CNI    → net2 구성
```

구성 요소

| 구성 요소 | 역할 |
| --- | --- |
| **Container Runtime** | Pod sandbox 네트워크 설정을 위해 CNI 호출 |
| **Multus** | 기본 네트워크와 요청된 추가 네트워크의 CNI 호출 조정 |
| **Primary CNI** | 기본 Pod 네트워크 구성 |
| **Delegate CNI** | Multus가 호출하는 실제 네트워크 플러그인 |
| **IPAM Plugin** | 해당 네트워크에서 사용할 IP 할당·회수 |
| **Kubernetes API** | Pod annotation과 NetworkAttachmentDefinition 저장·조회 |

### Pod 생성 흐름

1. 사용자가 추가 네트워크를 지정한 Pod 생성
2. 노드에 배정된 Pod에 대해 kubelet이 런타임에 sandbox 생성 요청
3. 런타임의 네트워크 설정 과정에서 Multus 호출
4. Multus가 기본 네트워크용 CNI를 호출하여 `eth0` 구성
5. Pod annotation에 지정된 NetworkAttachmentDefinition 확인
6. 해당 CNI 설정으로 추가 플러그인을 호출하여 `net1`, `net2` 구성
7. 설정 결과를 런타임에 반환하고 Pod의 네트워크 상태 annotation에 연결 정보 기록

## CNI Chaining

**하나의 네트워크 연결을 구성할 때 여러 CNI 플러그인을 순서대로 실행하여 기능을 조합하는 방식**

```
bridge       → 인터페이스와 bridge 연결 구성
   ↓ prevResult
tuning       → 해당 인터페이스의 MTU 등 조정
   ↓ prevResult
portmap      → hostPort 매핑 규칙 구성
```

- CNI 설정의 `plugins` 배열로 실행 순서 정의
- `ADD`에서는 앞 플러그인의 결과를 다음 플러그인에 `prevResult`로 전달
- `DEL`에서는 플러그인 목록을 역순으로 실행
- 플러그인의 역할에 따라 인터페이스 생성, 속성 변경, 규칙 추가 등을 수행

**플러그인의 실행 순서가 애플리케이션 패킷의 통과 순서를 뜻하는 것은 아니다.** `tuning`이 설정한 MTU나 `portmap`이 만든 규칙을 커널이 사용한다.

### 설정 예시

아래는 Chaining 구조를 보여주는 독립적인 `.conflist` 예시이다.

```json
{
  "cniVersion": "0.3.1",
  "name": "study-chain",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "br-study",
      "ipam": {
        "type": "host-local",
        "ranges": [[{ "subnet": "10.88.0.0/24" }]]
      }
    },
    {
      "type": "tuning",
      "mtu": 1400
    }
  ]
}
```

`bridge`가 연결을 만들고, `tuning`이 같은 연결의 인터페이스 속성을 조정한다. 이 구성에서 `tuning` 때문에 `net1`이 새로 생기는 것은 아니다.

### CNI Chaining vs Multi-Networking

| 구분 | CNI Chaining | Multus Multi-Networking |
| --- | --- | --- |
| 조합 단위 | 하나의 네트워크 설정 안의 플러그인 목록 | Pod에 부착할 여러 네트워크 연결 |
| 대표 목적 | 연결 생성 후 속성·정책·매핑 기능 추가 | 다른 네트워크용 인터페이스 추가 |
| 예시 | `bridge → tuning` | `eth0: Cilium`, `net1: macvlan` |
| 핵심 결과 | 기존 연결에 기능 조합 | 여러 네트워크에 연결된 Pod |

두 방식은 함께 사용할 수 있다.

```python
Multus
├─ Primary Network   → eth0
└─ Secondary Network → macvlan → tuning → net1
```

## NetworkAttachmentDefinition (NAD)

**Pod에 추가로 부착할 네트워크의 CNI 설정을 정의하는 Kubernetes Custom Resource**

```
NetworkAttachmentDefinition
   └─ 어떤 플러그인 / 부모 인터페이스 / IPAM을 사용할지 정의

Pod annotation
   └─ 어떤 NAD를 사용할지 선택

Multus
   └─ 선택된 설정으로 실제 CNI 호출
```

- **CRD**: `NetworkAttachmentDefinition`이라는 리소스 종류를 Kubernetes에 등록
- **NAD**: 그 종류로 만든 개별 네트워크 설정
- **Pod annotation**: Pod가 사용할 네트워크 설정 참조

NAD는 Namespace에 속한다. Namespace는 설정의 소속이며, 그 자체로 데이터 패스의 통신 격리를 보장하지는 않는다.

### macvlan NAD 예시

아래부터의 YAML과 IP는 **학습용 예시이며 실제 배포·실행 결과가 아니다.** 명령어는 Linux/Bash 기준이다.

- 사전 조건
    - Linux 워커에서 Primary CNI와 Multus가 구성되어 있음
    - NAD CRD와 `macvlan`, `host-local` 바이너리가 설치되어 있음
    - 예시 노드의 hostname label 값은 `worker1`
    - `worker1`의 `ens224`가 `192.168.100.0/24` 데이터망에 연결되어 있음
    - `192.168.100.100~120`은 기존 장비·DHCP가 사용하지 않는 예약 범위
    - 물리 스위치·가상 스위치가 Pod의 추가 MAC 주소 사용을 허용함

`macvlan-net.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: netlab
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-net
  namespace: netlab
spec:
  config: |
    {
      "cniVersion": "0.3.1",
      "name": "macvlan-net",
      "type": "macvlan",
      "master": "ens224",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "ranges": [[{
          "subnet": "192.168.100.0/24",
          "rangeStart": "192.168.100.100",
          "rangeEnd": "192.168.100.120",
          "gateway": "192.168.100.1"
        }]]
      }
    }
```

| 필드 | 의미 |
| --- | --- |
| `metadata.name` | Pod에서 참조할 NAD 이름 |
| `metadata.namespace` | NAD가 속한 Kubernetes Namespace |
| `spec.config` | **JSON 문자열**로 작성하는 CNI 설정 |
| `type` | 호출할 플러그인: `macvlan` |
| `master` | 노드의 부모 인터페이스: `ens224` |
| `mode` | macvlan 동작 모드 |
| `ipam.type` | IP 할당 방식 |
| `ranges` | 할당할 주소 범위 |

## macvlan / ipvlan / SR-IOV

### macvlan

**하나의 부모 인터페이스 위에 서로 다른 MAC 주소를 가진 가상 인터페이스를 만드는 Linux 네트워크 기능**

```
Node: ens224
       ├─ Pod A net1 → IP .100 / MAC A
       └─ Pod B net1 → IP .101 / MAC B
```

- 물리 네트워크에서는 Pod가 각자의 MAC·IP를 가진 장치처럼 보임
- 부모 인터페이스를 공유하며 Linux 커널의 macvlan 처리를 사용
- macvlan 자체가 VXLAN 같은 터널을 만드는 것은 아님
- `mode: bridge`는 같은 부모의 macvlan 인터페이스 간 로컬 통신을 허용

**macvlan의 `bridge` 모드가 별도의 Linux bridge 장치인 `cni0`를 만든다는 뜻은 아니다.**

#### 고려 사항

- NIC·스위치에서 여러 MAC 주소를 처리할 수 있어야 함
- VM 기반 Kubernetes에서는 가상 스위치의 MAC 필터링·위조 송신 제한도 확인
- 부모 인터페이스의 호스트 IP와 macvlan 자식 사이에는 직접 통신 제한이 있음
- 호스트 통신이 필요하면 Primary Network를 이용하거나, 호스트 측 macvlan 인터페이스와 경로를 별도로 설계

### ipvlan

**부모 인터페이스의 MAC 주소를 공유하면서, IP 주소를 기준으로 각 가상 인터페이스에 패킷을 전달하는 Linux 네트워크 기능**

```
Node: ens224 / MAC P
       ├─ Pod A net1 → IP .100 / MAC P
       └─ Pod B net1 → IP .101 / MAC P
```

| 모드 | 동작 | 고려 사항 |
| --- | --- | --- |
| **L2** | 같은 L2 네트워크에 연결, Broadcast·Multicast 사용 가능 | 외부에는 공유 MAC과 서로 다른 IP가 보임 |
| **L3** | Pod의 IP 처리를 부모 쪽 라우팅과 연결 | Pod 인터페이스에서 Broadcast·Multicast 송수신 불가, 외부 반환 경로 필요 |
| **L3S** | L3와 유사하며 conntrack 지원을 위한 대칭 처리 제공 | L3의 conntrack 제약을 보완 |
- 스위치 포트의 MAC 개수 제한이 있는 환경에서 유용
- CNI `ipvlan` 플러그인의 기본 모드는 `l2`
- Linux에서 `ip link`로 직접 만들 때의 기본값과 혼동하지 않도록 모드 명시 권장
- ipvlan도 부모 인터페이스의 호스트 IP와 직접 통신하는 데 제한이 있음

**MAC 주소를 공유해도 Pod IP를 같게 사용하는 것은 아니다.** IP는 각 엔드포인트를 구분할 수 있어야 한다.

### SR-IOV (Single Root I/O Virtualization)

**하나의 PCIe 장치를 여러 가상 PCIe Function으로 노출하여, 각각을 독립적인 장치처럼 사용할 수 있게 하는 하드웨어 기능**

| 개념 | 역할 |
| --- | --- |
| **PF (Physical Function)** | 물리 장치의 관리·설정을 담당하는 Function |
| **VF (Virtual Function)** | PF에서 제공하며 워크로드에 할당할 수 있는 가상 Function |

```
SR-IOV NIC / Port
└─ PF: 장치 관리
   ├─ VF 0 → Pod A
   ├─ VF 1 → Pod B
   └─ VF 2 → Pod C
```

VF들은 별도 PCI 주소를 가지지만 기반 물리 포트와 자원은 공유한다. VF를 할당했다고 Pod마다 독립적인 물리 회선이 생기는 것은 아니다.

#### Kubernetes 구성 요소

| 구성 요소 | 역할 |
| --- | --- |
| **호스트 설정 / Operator** | SR-IOV 활성화, VF 생성·드라이버 등 준비 |
| **SR-IOV Device Plugin** | 사용 가능한 장치를 kubelet에 등록하여 확장 리소스로 제공 |
| **Scheduler / kubelet** | Pod의 리소스 요청에 맞는 노드 선택 및 장치 할당 처리 |
| **Multus** | 할당된 VF의 `deviceID`를 네트워크 설정에 연결하여 CNI 호출 |
| **SR-IOV CNI** | 할당된 VF의 네트워크 연결과 MAC·VLAN·IP 등의 설정 수행 |

**Device Plugin의 장치 자원 제공과 CNI의 네트워크 설정은 서로 다른 역할이다.** VF 생성은 SR-IOV CNI가 담당하지 않는다.

#### Kernel Driver vs DPDK

```
Kernel Driver
애플리케이션 소켓 → Pod Linux 네트워크 스택 → VF → NIC → 외부망

DPDK / Userspace Driver
DPDK 애플리케이션 → VF의 패킷 큐를 사용자 공간에서 처리 → NIC → 외부망
```

- **SR-IOV 자체가 Linux 커널을 완전히 우회한다는 뜻은 아님**
- Kernel Driver 방식은 일반 소켓과 Pod의 Linux 인터페이스 사용
- DPDK 방식은 사용자 공간에서 장치 큐를 처리하도록 별도 드라이버·애플리케이션 구성
- DPDK용 VF에서는 일반적인 `ip addr` 인터페이스처럼 보이거나 IP가 설정된다고 가정하면 안 됨
- 성능은 NIC·드라이버·CPU·NUMA·패킷 크기와 워크로드에 따라 달라짐

### 비교

| 항목 | macvlan | ipvlan | SR-IOV |
| --- | --- | --- | --- |
| 구현 기반 | Linux 가상 인터페이스 | Linux 가상 인터페이스 | NIC의 PCIe 가상화 기능 |
| MAC | 자식 인터페이스마다 별도 MAC | 부모 MAC 공유 | 일반적인 Ethernet VF별 MAC 설정 가능 |
| 전달 기준 | MAC 기반 인터페이스 구분 | IP 기반 인터페이스 구분 | NIC의 VF 분류·전달 기능 |
| 하드웨어 요구 | SR-IOV NIC 불필요 | SR-IOV NIC 불필요 | SR-IOV 지원 NIC·환경 필요 |
| 대표 목적 | 기존 L2망에 Pod 연결 | 공유 MAC으로 추가 IP 연결 | 가상 스위칭 경로를 줄인 통신 |
| 주요 확인점 | 추가 MAC 허용, 호스트 직접 통신 제한 | 모드별 라우팅·Broadcast 특성 | VF 수, 장치 할당, 드라이버, 정책 경로 |
- 같은 부모 인터페이스에 macvlan과 ipvlan을 동시에 붙일 수는 없다.

## Multi-NIC Pod Packet Flow

아래는 **IPv4·일반 소켓 통신·별도 NAT 없음**을 가정한 설명용 흐름이다. 기본 네트워크의 상세 처리는 사용하는 Primary CNI에 따라 달라진다.

### 라우팅: 어느 인터페이스로 나갈까?

예를 들어 Flannel 기반 Pod A에 다음 경로가 있다고 가정한다. 실제 Primary 경로는 CNI 설정에 따라 다르다.

```
default via 10.244.1.1 dev eth0
10.244.1.0/24 dev eth0 src 10.244.1.10
192.168.100.0/24 dev net1 src 192.168.100.100
```

| 목적지 | 선택 경로 | 이유 |
| --- | --- | --- |
| `10.244.1.11` | `eth0` | Primary의 연결 대역 |
| `192.168.100.101` | `net1` | Secondary의 연결 대역 |
| `192.168.100.50` | `net1` | 같은 데이터망의 외부 장비 |
| 그 외 주소 | 이 예시에서는 `eth0` | 일치하는 상세 경로가 없으면 default route |

일반적인 목적지 기반 라우팅은 **더 구체적인 Prefix를 우선**한다. `net1`이 있다는 이유만으로 모든 외부 트래픽이 `net1`으로 나가지는 않는다. Policy routing이나 애플리케이션의 주소·장치 바인딩이 있으면 그것도 함께 확인한다.

```bash
# 실제 Pod B의 net1 IP로 바꿔 확인
kubectl -n netlab exec multi-nic-a -- ip -4 route get 192.168.100.101
kubectl -n netlab exec multi-nic-a -- ip rule
```

### 1. Primary Network로 Pod 간 통신

```
Pod A: eth0 10.244.1.10
   ↓
Primary CNI가 구성한 데이터 패스
   ↓
Pod B: eth0 10.244.1.11
```

- Flannel이면 bridge·라우팅·필요한 경우 VXLAN 등의 경로
- Calico면 선택한 라우팅·터널·정책 경로
- Cilium이면 해당 구성의 eBPF·라우팅·터널 경로

Secondary 인터페이스를 추가해도 이 기본 연결은 계속 존재한다. 다만 default route를 변경하면 기존 경로에 영향을 줄 수 있다.

### 2. 같은 노드의 macvlan Pod ↔ Pod

조건: 같은 부모 `ens224`, 같은 서브넷, `mode: bridge`.

```
Pod A net1 (.100 / MAC A)
          ↓
Linux macvlan 로컬 전달
          ↓
Pod B net1 (.101 / MAC B)
```

1. Pod A가 `192.168.100.101`로 패킷 생성
2. 라우팅 조회로 `net1` 선택
3. ARP로 목적지 Pod B의 MAC 확인
4. Ethernet 헤더는 `src=MAC A`, `dst=MAC B`
5. macvlan이 같은 부모 아래의 목적지 인터페이스로 로컬 전달
6. Pod B의 `net1`에서 수신

유니캐스트 데이터 패킷은 외부 스위치까지 나갈 필요가 없다. ARP 같은 Broadcast는 별도로 외부에 보일 수 있다.

### 3. 다른 노드의 macvlan Pod ↔ Pod

조건: 두 노드의 부모 NIC가 **같은 L2/VLAN**에 연결되어 있고 IP가 중복되지 않음. 앞의 단일 노드 실습을 확장한 설명이다.

```
Node 1                                    Node 2
Pod A net1 (.100 / MAC A)                  Pod B net1 (.101 / MAC B)
        │                                          ▲
     macvlan                                    macvlan
        │                                          │
      ens224 ─────── 물리 스위치 / 같은 VLAN ─────── ens224
```

1. Pod A가 `net1`으로 ARP 요청, Pod B의 MAC 확인
2. Node 1의 부모 NIC로 프레임 송신
3. 스위치가 목적지 MAC B가 있는 포트로 전달
4. Node 2가 수신한 프레임을 macvlan으로 분류
5. Pod B의 `net1`으로 전달

```
IP:       192.168.100.100 → 192.168.100.101
Ethernet: MAC A           → MAC B
```

이 구성에는 macvlan이 추가하는 VXLAN 헤더가 없다. **같은 IP 서브넷을 할당하는 것만으로 서로 다른 L2망이 연결되지는 않는다.**

### 4. net1 → 다른 서브넷의 외부 장비

외부 장비가 `192.168.200.50`이라면 연결 대역 경로만으로는 부족하다. 데이터망의 실제 라우터 `192.168.100.1`로 가는 경로를 추가한다고 가정한다.

NAD의 IPAM 설정에 다음 `routes`를 포함할 수 있다.

```json
"routes": [
  { "dst": "192.168.200.0/24", "gw": "192.168.100.1" }
]
```

```
Pod A net1        → 데이터망 Gateway → 외부 장비
192.168.100.100      192.168.100.1       192.168.200.50
```

1. `192.168.200.0/24` 경로를 선택하여 `net1`으로 송신
2. ARP는 최종 목적지 대신 **다음 홉 `192.168.100.1`*의 MAC을 조회
3. 첫 링크에서 목적지 IP는 `.200.50`, 목적지 MAC은 Gateway MAC
4. Gateway가 다음 링크로 라우팅하고 Ethernet 헤더를 새로 구성
5. 외부 장비가 Pod 대역으로 응답할 수 있는 반환 경로도 필요

```
Pod에서 첫 링크로 나갈 때

IP:       192.168.100.100 → 192.168.200.50
Ethernet: MAC A           → Gateway MAC
```

**목적지 IP는 최종 장비, 목적지 MAC은 현재 링크의 다음 홉을 가리킨다.** 기존 Pod에 NAD를 수정하는 것만으로 경로가 즉시 재설정된다고 가정하지 않고, 일반적인 생성 시 적용 모델에서는 Pod를 재생성해 확인한다.

### 5. ipvlan L2의 수신 흐름

```
외부 장비
  │ dst IP = Pod B IP / dst MAC = Node 2의 부모 MAC
  ▼
Node 2 부모 NIC
  ▼
Linux ipvlan이 목적지 IP를 확인
  ▼
Pod B net1
```

macvlan이 Pod별 MAC을 사용하는 것과 달리, ipvlan은 공유 MAC으로 받은 패킷을 **목적지 IP**로 구분한다. 이 설명은 L2 모드 기준이며 L3 모드는 부모 쪽 라우팅과 외부 반환 경로 설계가 추가로 필요하다.

### 6. SR-IOV VF의 통신 흐름

```
Pod A 애플리케이션
      ↓
Pod의 VF 인터페이스 / Kernel Driver
      ↓
NIC 하드웨어의 VF 처리
      ↓
물리 포트 → 외부 스위치 → 목적지
```

일반적인 VF 직접 연결 구성에서는 Primary의 veth·Linux bridge 경로를 사용할 필요가 없다. 하지만 **Kernel Driver를 사용하면 Pod의 Linux 네트워크 스택은 사용한다.** Switchdev·representor·하드웨어 오프로딩을 적용한 구성은 전달·정책 경로가 달라질 수 있다.

---

- 참고자료
    
    https://github.com/k8snetworkplumbingwg/multus-cni
    
    https://docs.kernel.org/PCI/pci-iov-howto.html
    
    https://k8snetworkplumbingwg.github.io/multus-cni/docs/how-to-use.html
    
    https://www.cni.dev/plugins/current/main/macvlan/
    
    https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin
