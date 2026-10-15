# Week 6. Multi-Networking & Secondary Networking

## 1. 학습 목표

Kubernetes Pod에 기본 Network 외의 Network Interface를 추가하는 이유와, Multus가 여러 CNI Plugin을 연결하는 방식을 이해한다.

- Primary Network와 Secondary Network의 역할은 무엇인가?
- Multus와 `NetworkAttachmentDefinition`은 어떻게 연결되는가?
- macvlan, ipvlan, SR-IOV는 어떤 차이가 있는가?
- 여러 Interface를 가진 Pod의 Packet은 어느 경로로 전달되는가?

> Multus는 여러 Network Interface를 연결하도록 조정하는 **CNI Meta Plugin**이다. 실제 Interface와 Data Path는 Multus가 호출한 macvlan, ipvlan, SR-IOV 등의 CNI Plugin이 구성한다.

---

## 2. 이전 주차와 연결

앞선 주차에서는 하나의 기본 Pod Network를 중심으로 통신 경로를 살펴봤다.

```text
Week 2
  CNI가 Pod의 Network Namespace에 Interface와 IP, Route를 구성

Week 3~5
  Flannel, Calico, Cilium이 Primary Network를 구현하는 방식 비교

Week 6
  Primary Network는 유지하면서 Pod에 Secondary Network 추가
```

일반적인 Pod에는 `lo`와 Primary Network의 `eth0`이 있다. Multus를 사용하면 여기에 `net1`, `net2`와 같은 Interface를 더 연결할 수 있다.

```text
Pod Network Namespace

lo
eth0  ── Primary Network ── Kubernetes Pod·Service Network
net1  ── Secondary Network A ── 외부 L2 Network
net2  ── Secondary Network B ── 고성능 SR-IOV Network
```

Primary Network는 Kubernetes의 기본 Pod-to-Pod 통신을 제공하며, Secondary Network는 특정 업무를 위한 별도 경로를 추가한다. Secondary Network가 Primary Network를 자동으로 대체하는 것은 아니다. [Multus Quickstart](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/quickstart.md)

---

## 3. Multi-Network가 필요한 상황

하나의 Network만으로도 일반적인 Application은 실행할 수 있다. Multi-Network는 Traffic의 목적이나 요구사항이 서로 다를 때 사용한다.

| 상황 | 구성 예시 | 목적 |
| --- | --- | --- |
| 관리 Traffic과 Data Traffic 분리 | `eth0`: Cluster Network, `net1`: Data Network | 장애 범위와 운영 경로 분리 |
| 기존 사내·물리 Network 연결 | `net1`: macvlan 또는 ipvlan | Pod를 기존 L2/L3 Network에 직접 연결 |
| Network Function 실행 | 여러 Secondary Interface | 방화벽, Router, 5G Core 등의 입·출력 Network 분리 |
| 낮은 지연과 높은 처리량 필요 | `net1`: SR-IOV VF | 가상 Switch 경로를 줄여 NIC 자원을 직접 활용 |
| 서로 다른 보안 영역 연결 | 업무망과 관리망을 별도 Interface로 구성 | Network별 접근 경로 분리 |

Interface가 늘어나면 Route, IPAM, MTU, Policy와 장애 지점도 함께 늘어난다. 따라서 단순한 Application이라면 Primary Network만 사용하고, 분리 목적이 명확할 때 Secondary Network를 추가하는 편이 좋다.

---

## 4. Multus Architecture

Multus는 Container Runtime과 실제 CNI Plugin 사이에서 여러 Network 연결을 조정한다.

```text
Kubernetes API
  ├─ Pod
  │    └─ networks Annotation: data-net, sriov-net
  └─ NetworkAttachmentDefinition
       ├─ data-net  → macvlan 설정
       └─ sriov-net → SR-IOV 설정
                ▲
                │ 조회
Node            │
  Container Runtime
        │ CNI ADD / DEL
        ▼
      Multus (Meta Plugin)
        ├─ Primary CNI 호출   → eth0
        ├─ macvlan CNI 호출   → net1
        └─ SR-IOV CNI 호출    → net2
```

Multus가 하는 일은 다음과 같다.

1. Runtime의 CNI 요청을 받는다.
2. Primary CNI를 호출해 기본 Network를 연결한다.
3. Pod Annotation에서 요청한 Network를 확인한다.
4. Kubernetes API에서 해당 `NetworkAttachmentDefinition`을 조회한다.
5. 정의에 적힌 CNI Plugin을 호출해 Secondary Interface를 추가한다.
6. 연결 결과를 Pod의 `network-status` Annotation에 기록한다.

Multus 4.0부터는 `thin`과 `thick` 배포 방식이 있다. 공식 문서는 일반적인 환경에 `thick` 방식을 권장한다.

| 방식 | 구조 | 특징 |
| --- | --- | --- |
| Thin Plugin | Runtime이 Multus Binary를 직접 실행 | 구조가 단순하고 자원 사용량이 적음 |
| Thick Plugin | Runtime → `multus-shim` → Unix Socket → `multus-daemon` | Node별 Daemon이 NAD 조회와 Delegate 호출을 담당하고 Metrics 등의 기능 제공 |

```text
Thick Plugin

Runtime
  → multus-shim
  → Unix Domain Socket
  → multus-daemon
  → Primary / Secondary CNI Plugin
```

`multus-daemon`은 각 Node에서 동작하며 Control Plane처럼 연결을 조정한다. 실제 Packet이 매번 이 Daemon을 통과하는 것은 아니다. Pod 생성 시 구성된 Interface와 Linux Network 경로를 통해 Packet이 전달된다. [Multus Architecture](https://github.com/k8snetworkplumbingwg/multus-cni), [Multus Thick Plugin](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/thick-plugin.md)

---

## 5. CNI Chaining과 여러 Network 연결

두 개념은 비슷해 보이지만 구분해야 한다.

### CNI Chaining

하나의 Network Attachment에 여러 Plugin을 순서대로 실행하는 방식이다. 앞 Plugin의 실행 결과가 `prevResult`로 다음 Plugin에 전달된다.

```text
하나의 eth0 구성

bridge CNI
  → portmap CNI
  → bandwidth CNI
```

`ADD`는 설정된 순서대로 실행하고, `DEL`은 반대 순서로 실행한다.

### Multi-Network Attachment

하나의 Pod에 서로 다른 Network Attachment를 여러 개 만드는 방식이다.

```text
eth0 → Primary CNI
net1 → macvlan CNI
net2 → SR-IOV CNI
```

Multus는 여러 Attachment를 조정하는 Meta Plugin이며, 각각의 Attachment 내부에서도 CNI Plugin Chain을 사용할 수 있다. 즉 **Plugin 여러 개가 하나의 Interface를 함께 완성하는 것**과 **Pod에 Interface 여러 개를 만드는 것**은 같은 의미가 아니다. [CNI Specification](https://www.cni.dev/docs/spec/)

---

## 6. Primary Network와 Secondary Network

| 구분 | Primary Network | Secondary Network |
| --- | --- | --- |
| 대표 Interface | 보통 `eth0` | 보통 `net1`, `net2` |
| 연결 여부 | 모든 Pod에 기본 연결 | 요청한 Pod에만 연결 |
| 주요 역할 | Kubernetes Pod-to-Pod, Service, DNS, API 접근 | 외부 L2/L3망, 전용 Data Plane, 특수 NIC 연결 |
| 설정 위치 | Cluster의 기본 CNI 설정 | `NetworkAttachmentDefinition`과 Pod Annotation |
| 구현 예시 | Calico, Cilium, Flannel | macvlan, ipvlan, SR-IOV, bridge 등 |

Kubernetes가 인식하는 Pod IP와 기본 Service 경로는 일반적으로 Primary Network를 기준으로 한다. Secondary Interface가 생겼다고 해서 해당 IP가 자동으로 일반 Kubernetes Service의 Endpoint가 되는 것은 아니다.

---

## 7. NetworkAttachmentDefinition

`NetworkAttachmentDefinition`, 줄여서 NAD는 Secondary Network의 CNI 설정을 Kubernetes Custom Resource로 저장한다.

다음은 `eth1`을 Parent Interface로 사용하는 macvlan Network 예시다.

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: data-net
spec:
  config: |-
    {
      "cniVersion": "1.0.0",
      "name": "data-net",
      "type": "macvlan",
      "master": "eth1",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.100.0/24",
        "rangeStart": "192.168.100.100",
        "rangeEnd": "192.168.100.200",
        "gateway": "192.168.100.1"
      }
    }
```

주요 필드는 다음과 같다.

| 필드 | 의미 |
| --- | --- |
| `metadata.name` | Pod가 Annotation에서 참조할 Network 이름 |
| `type` | Multus가 호출할 CNI Plugin Binary |
| `master` | macvlan·ipvlan이 연결될 Node의 Parent Interface |
| `mode` | Plugin이 사용할 동작 Mode |
| `ipam` | Pod Interface에 IP를 할당할 방법 |

Pod는 `k8s.v1.cni.cncf.io/networks` Annotation으로 NAD를 요청한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-network-pod
  annotations:
    k8s.v1.cni.cncf.io/networks: |-
      [
        { "name": "data-net", "interface": "data0" }
      ]
spec:
  containers:
    - name: app
      image: alpine
      command: ["sh", "-c", "sleep infinity"]
```

Pod 안에는 보통 다음 Interface가 만들어진다.

```text
lo      Loopback
eth0    Primary Network
data0   data-net NAD로 요청한 Secondary Network
```

Interface 이름을 요청하지 않으면 Multus가 `net1`, `net2`와 같은 이름을 사용할 수 있다. 연결 결과는 다음 명령으로 확인할 수 있다.

```bash
kubectl exec multi-network-pod -- ip addr
kubectl exec multi-network-pod -- ip route
kubectl get pod multi-network-pod \
  -o jsonpath='{.metadata.annotations.k8s\.v1\.cni\.cncf\.io/network-status}'
```

`host-local` IPAM은 할당 정보를 Node Local Database에 저장한다. 여러 Node가 같은 주소 범위를 독립적으로 사용하면 IP가 중복될 수 있으므로, 실제 Multi-Node 환경에서는 DHCP나 Cluster 전체 할당 상태를 관리하는 Whereabouts 같은 IPAM을 검토해야 한다. [Multus Quickstart](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/quickstart.md), [Whereabouts](https://github.com/k8snetworkplumbingwg/whereabouts)

---

## 8. Pod가 생성될 때의 흐름

```text
1. Scheduler가 Pod를 Node에 배치
2. Runtime이 Pod Network Namespace 생성
3. Runtime이 Multus에 CNI ADD 요청
4. Multus가 Primary CNI 호출
   → eth0, Pod IP, 기본 Route 구성
5. Multus가 Pod의 networks Annotation 확인
6. Kubernetes API에서 NAD 조회
7. NAD마다 해당 CNI Plugin 호출
   → net1, net2와 IP·Route 구성
8. Multus가 전체 결과를 Runtime에 반환
9. network-status Annotation에서 연결 결과 확인
```

Pod가 삭제되면 `DEL` 요청으로 각 Plugin이 Interface와 IPAM 할당 등의 자원을 정리한다. CNI Spec은 같은 Attachment를 `(CNI_CONTAINERID, CNI_IFNAME)` 조합으로 식별한다. 따라서 같은 Network를 두 번 연결하려면 서로 다른 Interface 이름이 필요하다. [CNI Specification](https://www.cni.dev/docs/spec/)

---

## 9. macvlan

macvlan은 하나의 Parent Interface 아래에 서로 다른 MAC 주소를 가진 Virtual Interface를 만든다.

```text
Node eth1 (Parent Interface)
  ├─ Pod A net1: IP 192.168.100.10 / MAC A
  └─ Pod B net1: IP 192.168.100.11 / MAC B
             │
             ▼
       기존 L2 Network
```

| 항목 | 내용 |
| --- | --- |
| 식별 방식 | Pod Interface마다 별도 MAC과 IP 사용 |
| 대표 Mode | `bridge`, `private`, `vepa`, `passthru` |
| 장점 | 기존 L2 Network와 DHCP에 연결하기 쉬움, 구조가 비교적 단순함 |
| 주의점 | Switch가 많은 MAC 주소를 학습해야 함, 일부 무선 NIC에서 사용하기 어려움 |
| 적합한 경우 | Pod를 기존 VLAN·L2 Network의 독립된 장비처럼 연결할 때 |

기본값인 `bridge` Mode에서는 같은 Parent에 연결된 macvlan Interface끼리 통신할 수 있다. 실제 Network에서는 Switch의 Port Security나 MAC 주소 수 제한도 확인해야 한다. [CNI macvlan Plugin](https://www.cni.dev/plugins/current/main/macvlan/)

---

## 10. ipvlan

ipvlan은 macvlan처럼 Parent Interface를 가상화하지만, Virtual Interface들이 Parent와 같은 MAC 주소를 공유한다. Kernel은 목적지 IP를 기준으로 어느 Virtual Interface에 Packet을 전달할지 결정한다.

```text
Node eth1: MAC X
  ├─ Pod A net1: IP 192.168.100.10 / MAC X
  └─ Pod B net1: IP 192.168.100.11 / MAC X
```

| 항목 | 내용 |
| --- | --- |
| 식별 방식 | 여러 Interface가 MAC을 공유하고 서로 다른 IP 사용 |
| 대표 Mode | `l2`, `l3`, `l3s` |
| 장점 | 외부 Switch에 많은 MAC 주소를 노출하지 않음 |
| 주의점 | 공식 CNI Plugin 기준으로 Virtual Interface와 Master Interface가 직접 통신할 수 없음 |
| 적합한 경우 | Switch의 MAC 수 제약이 있거나 IP 중심으로 Network를 구성할 때 |

모든 ipvlan Interface가 같은 MAC을 공유하므로 공식 CNI DHCP Plugin은 현재 필요한 Client ID 방식을 지원하지 않는다. 일반적으로 별도의 IPAM 방식을 함께 고려한다. [CNI ipvlan Plugin](https://www.cni.dev/plugins/current/main/ipvlan/)

---

## 11. SR-IOV

SR-IOV는 하나의 Physical Function을 여러 Virtual Function으로 나누고, VF를 Pod에 할당하는 방식이다.

```text
Physical NIC
  └─ PF (Physical Function)
       ├─ VF 0 ── Pod A net1
       ├─ VF 1 ── Pod B net1
       └─ VF 2 ── 할당 가능
```

Kubernetes에서는 CNI Plugin만으로 VF의 Scheduling까지 처리할 수 없기 때문에 여러 구성요소가 협력한다.

```text
SR-IOV Device Plugin
  → Node의 VF 탐색
  → kubelet에 Extended Resource로 광고

Scheduler / kubelet
  → Pod가 요청한 VF Resource를 가진 Node 선택
  → 사용 가능한 VF 할당

Multus
  → NAD와 할당된 Device ID를 연결
  → SR-IOV CNI 호출

SR-IOV CNI
  → VF를 Pod Network Namespace에 연결
  → MAC, VLAN, IP 등 구성
```

NAD에는 Device Plugin의 Resource 이름을 연결한다.

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: sriov-net
  annotations:
    k8s.v1.cni.cncf.io/resourceName: example.com/sriov_net
spec:
  config: |-
    {
      "cniVersion": "1.0.0",
      "name": "sriov-net",
      "type": "sriov",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.200.0/24"
      }
    }
```

Pod는 같은 Extended Resource를 요청한다.

```yaml
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: sriov-net
spec:
  containers:
    - name: app
      resources:
        limits:
          example.com/sriov_net: 1
```

| 장점 | 주의점 |
| --- | --- |
| 낮은 지연과 높은 처리량 | SR-IOV를 지원하는 NIC와 Driver 필요 |
| Virtual Switch 경로와 CPU Overhead를 줄일 수 있음 | 만들 수 있는 VF 수가 제한됨 |
| VF별 MAC, VLAN 등의 설정 가능 | Device Plugin, CNI, Firmware와 Node 설정까지 함께 관리해야 함 |
| DPDK 같은 User Space Networking과 결합 가능 | 일반 veth 기반 Network보다 운영과 장애 분석이 복잡함 |

Device Plugin이 VF를 **발견하고 할당 가능한 Resource로 광고**한다면, SR-IOV CNI는 할당된 VF를 **Pod에 연결하고 Network를 설정**한다. 두 역할을 혼동하지 않아야 한다. [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/), [SR-IOV Device Plugin](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin), [SR-IOV CNI](https://github.com/k8snetworkplumbingwg/sriov-cni)

---

## 12. macvlan, ipvlan, SR-IOV 비교

| 구분 | macvlan | ipvlan | SR-IOV |
| --- | --- | --- | --- |
| 연결 방식 | Parent 아래 별도 MAC의 Virtual Interface | Parent MAC을 공유하는 Virtual Interface | NIC의 VF를 Pod에 할당 |
| 외부에 보이는 MAC | Pod마다 다름 | Parent MAC 공유 | 일반적으로 VF마다 존재 |
| Hardware 의존성 | 낮음 | 낮음 | 높음 |
| 성능 특성 | 일반 Kernel Network 경로 | 일반 Kernel Network 경로 | Hardware 기능을 활용한 고성능 경로 |
| 운영 난이도 | 보통 | 보통 | 높음 |
| 대표 사용 사례 | 기존 L2·DHCP Network 연결 | MAC 수를 줄인 L2/L3 연결 | Telco, NFV, DPDK, 고성능 Data Plane |

선택 기준을 간단히 정리하면 다음과 같다.

```text
기존 L2 Network에 Pod별 MAC이 필요하다
  → macvlan

외부 Switch에 많은 MAC을 노출하기 어렵다
  → ipvlan

Hardware 직접 활용과 높은 성능이 가장 중요하다
  → SR-IOV
```

---

## 13. Multi-NIC Pod의 Packet Flow

다음과 같은 Pod를 가정한다.

```text
eth0: 10.244.1.10/24       Primary Network
net1: 192.168.100.10/24    Secondary Network
```

Route가 다음과 같다면 목적지에 따라 Interface가 선택된다.

```text
default via 10.244.1.1 dev eth0
10.244.0.0/16 dev eth0
192.168.100.0/24 dev net1
```

```text
목적지 10.244.2.20
  → 10.244.0.0/16 Route 일치
  → eth0

목적지 192.168.100.50
  → 192.168.100.0/24 Route 일치
  → net1

그 외 목적지
  → default Route
  → eth0
```

Packet이 전송될 때마다 Multus가 경로를 선택하는 것이 아니다. Interface와 Route 구성이 끝난 뒤에는 Linux Routing Table이 가장 구체적으로 일치하는 Prefix를 우선해 경로를 선택한다.

### 다른 Node의 Secondary Network Pod로 통신

macvlan `bridge` Mode로 같은 외부 L2 Network에 연결했다고 가정한다.

```text
Pod A net1
  → Node A의 macvlan Parent Interface
  → 물리 Switch / VLAN
  → Node B의 macvlan Parent Interface
  → Pod B net1
```

이 경로는 Primary CNI의 Overlay를 반드시 통과하는 것이 아니다. 대신 두 Node의 Parent NIC가 같은 VLAN에 연결되어 있는지, Switch가 해당 MAC을 전달하는지, Secondary IP가 중복되지 않는지를 확인해야 한다.

### Route 구성 시 주의할 점

- Secondary Network에도 Default Route를 추가하면 기존 `eth0`의 Default Route와 경쟁할 수 있다.
- 요청은 `net1`으로 나갔는데 응답이 `eth0`으로 들어오는 비대칭 Routing이 생길 수 있다.
- Interface별로 출발지 주소까지 구분해야 한다면 Policy Routing이 필요할 수 있다.
- Primary와 Secondary Network의 MTU가 다르면 Fragmentation이나 Packet Drop을 확인해야 한다.
- NetworkPolicy 적용 여부는 각 Network Plugin과 Policy 구현에 따라 다르다. 기본 NetworkPolicy가 Secondary Interface까지 자동 보호한다고 가정하면 안 된다.

Kubernetes NetworkPolicy는 이를 지원하는 Network Plugin이 있어야 실제로 적용된다. NAD 기반 Network를 위한 별도의 MultiNetworkPolicy 프로젝트도 있지만 현재 공식 저장소에서는 개발 중인 API로 설명한다. [Kubernetes NetworkPolicy](https://kubernetes.io/docs/reference/kubernetes-api/networking/network-policy-v1/), [Multi-NetworkPolicy](https://github.com/k8snetworkplumbingwg/multi-networkpolicy)

---

## 14. 전체 흐름 연결

```text
1. Cluster 관리자가 Primary CNI와 Multus 설치
2. 관리자가 macvlan·ipvlan·SR-IOV 등의 NAD 생성
3. 사용자가 Pod Annotation에서 필요한 NAD 요청
4. Runtime이 Multus에 CNI ADD 호출
5. Multus가 Primary CNI를 호출해 eth0 구성
6. Multus가 NAD를 조회하고 Secondary CNI를 호출해 net1 구성
7. Pod가 Linux Route에 따라 eth0 또는 net1로 Packet 전송
8. Pod 삭제 시 CNI DEL로 Interface와 IPAM·Device 자원 정리
```

```text
Kubernetes API의 설정
  Pod Annotation + NAD
           │
           ▼
Pod 생성 시 구성
  Multus + 실제 CNI Plugin
           │
           ▼
실행 중 Packet 처리
  Interface + Linux Route + 각 Plugin의 Data Path
```

Multus는 **어떤 Network를 Pod에 붙일지 조정**하고, 실제 CNI Plugin은 **어떤 Interface와 Data Path를 만들지 구현**하며, Linux는 **구성된 Route에 따라 실행 중인 Packet의 경로를 선택**한다.

---

## 15. 학습하면서 생긴 질문

### Q1. Pod에 `eth0`과 `net1`이 있으면 Packet은 어느 Interface로 나갈까?

Multus가 Packet마다 Interface를 선택하지 않는다. Linux Routing Table에서 목적지 주소와 가장 길게 일치하는 Prefix의 Route가 선택된다.

```text
192.168.100.0/24 dev net1
default via 10.244.1.1 dev eth0
```

이 경우 `192.168.100.50`은 `net1`으로 나가고, 별도 Route가 없는 목적지는 `eth0`의 Default Route를 사용한다. 여러 Interface가 같은 목적지 Prefix나 Default Route를 가진다면 Metric과 Policy Routing까지 확인해야 한다.

### Q2. macvlan NAD에서 `host-local` IPAM을 사용하면 Multi-Node에서도 IP가 중복되지 않을까?

중복될 수 있다. `host-local`은 할당 상태를 각 Node의 Local Database에서 관리하므로 다른 Node가 같은 주소 범위에서 동일한 IP를 선택할 가능성이 있다.

```text
Node A의 host-local → 192.168.100.100 할당
Node B의 host-local → 192.168.100.100 할당 가능
```

Multi-Node에서 하나의 Secondary Subnet을 공유한다면 Node마다 겹치지 않는 Range를 배정하거나, DHCP 또는 Cluster 전체 할당을 조정하는 Whereabouts 같은 IPAM을 사용할 수 있다.

### Q3. Kubernetes NetworkPolicy를 만들면 Secondary Interface도 보호될까?

항상 그렇지는 않다. NetworkPolicy는 이를 구현하는 Network Plugin이 실제 Data Plane에 규칙을 적용해야 동작한다. Primary CNI가 `eth0`에 적용한 Policy가 별도 Plugin으로 만든 `net1`까지 자동으로 적용된다고 가정할 수 없다. Secondary Network의 Plugin과 Policy 지원 범위를 별도로 확인해야 한다.

---

## 16. 용어 설명집

| 용어 | 설명 |
| --- | --- |
| Multi-Homed Pod | 두 개 이상의 Network Interface를 가진 Pod |
| Primary Network | Kubernetes의 기본 Pod·Service 통신을 제공하는 Network |
| Secondary Network | 특정 목적을 위해 Pod에 추가로 연결한 Network |
| Multus | 여러 CNI Plugin과 Network Attachment를 조정하는 CNI Meta Plugin |
| Meta Plugin | 다른 CNI Plugin을 호출하고 실행을 조정하는 Plugin |
| Delegate | Multus가 실제 Network 구성을 맡기기 위해 호출하는 CNI 설정 또는 Plugin |
| CNI Chaining | 하나의 Attachment에서 여러 Plugin을 순서대로 실행하는 방식 |
| `prevResult` | CNI Chain에서 앞 Plugin의 실행 결과를 다음 Plugin에 전달하는 값 |
| NAD | `NetworkAttachmentDefinition`의 약자. Secondary Network의 CNI 설정을 저장하는 Custom Resource |
| IPAM | IP Address Management. Interface에 사용할 IP 주소를 할당하고 회수하는 기능 |
| Parent / Master Interface | macvlan·ipvlan Virtual Interface의 기반이 되는 Node Interface |
| macvlan | Parent 아래에 서로 다른 MAC을 가진 Virtual Interface를 만드는 방식 |
| ipvlan | Parent MAC을 공유하고 IP를 기준으로 Packet을 구분하는 방식 |
| SR-IOV | 하나의 PCIe Device를 여러 Virtual Function으로 나누어 사용하는 기술 |
| PF | Physical Function. SR-IOV NIC의 물리 기능을 관리하는 Function |
| VF | Virtual Function. PF에서 생성되어 Pod에 할당할 수 있는 경량 PCIe Function |
| Device Plugin | Node의 특수 Hardware를 발견하고 kubelet에 Extended Resource로 광고하는 Plugin |
| Extended Resource | `vendor-domain/resource` 형식으로 Kubernetes에 등록하는 사용자 정의 Node Resource |
| Policy Routing | 목적지뿐 아니라 출발지 주소, Mark 등의 조건을 이용해 Routing Table을 선택하는 방식 |

---

## 17. 참고 자료

- [Multus CNI](https://github.com/k8snetworkplumbingwg/multus-cni)
- [Multus Quickstart](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/quickstart.md)
- [Multus Configuration Reference](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/configuration.md)
- [Multus Thick Plugin](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/thick-plugin.md)
- [CNI Specification](https://www.cni.dev/docs/spec/)
- [CNI macvlan Plugin](https://www.cni.dev/plugins/current/main/macvlan/)
- [CNI ipvlan Plugin](https://www.cni.dev/plugins/current/main/ipvlan/)
- [Network Plumbing Working Group Multi-Network CRD Specification](https://github.com/k8snetworkplumbingwg/multi-net-spec)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
- [SR-IOV Device Plugin](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin)
- [SR-IOV CNI](https://github.com/k8snetworkplumbingwg/sriov-cni)
- [Whereabouts IPAM](https://github.com/k8snetworkplumbingwg/whereabouts)
- [Kubernetes NetworkPolicy](https://kubernetes.io/docs/reference/kubernetes-api/networking/network-policy-v1/)
