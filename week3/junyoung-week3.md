# Week 3. CNI Overview & Cloud Native Networking Landscape

> CNI가 "무엇을 표준화했고 무엇을 표준화하지 않았는지"를 구분하고, 그 빈칸을 Flannel / Calico / Cilium이 각각 어떻게 다르게 채웠는지 비교해서 이해

2주차에서는 Pod가 IP를 받고 Service 요청이 Pod에 도착하기까지의 **경로**를 봤다. 3주차는 그 경로를 실제로 만드는 **규약(CNI)과 구현체**를 본다.

---

## 1. CNI의 역할과 구조

### 1.1 CNI는 어디까지 정하는 규약인가

CNI(Container Network Interface)는 컨테이너 런타임과 네트워크 플러그인 사이의 규약이다. 스펙 문서와 참조 라이브러리(libcni), 참조 플러그인 모음으로 구성된다.

| 구분 | 내용 |
| --- | --- |
| CNI가 정하는 것 | 플러그인을 **언제** 호출하는지(생성/삭제 등), **어떤 형식**으로 입력을 주고받는지, 결과 JSON의 구조 |
| CNI가 정하지 않는 것 | 어떤 방식으로 네트워크를 구성할지(VXLAN·BGP·eBPF 등), NetworkPolicy, Service 구현, 암호화, 멀티클러스터 |

즉 CNI는 **"연결해 달라"는 요청 방식만 표준화**한 것이고, 그 요청을 받은 뒤의 방법은 전적으로 구현체 몫이다. Flannel과 Cilium이 완전히 다른 동작을 하면서도 같은 런타임에서 교체 가능한 이유가 여기에 있다.

### 1.2 실행 모델 — 데몬이 아니라 실행 파일

CNI 플러그인은 상주 데몬이 아니라 **짧게 실행되고 끝나는 실행 파일**이다. 런타임이 이를 실행하면서 필요한 정보를 넘긴다.

| 경로 / 채널 | 역할 |
| --- | --- |
| `/opt/cni/bin/` | 플러그인 실행 파일이 놓이는 기본 위치 |
| `/etc/cni/net.d/*.conflist` | 네트워크 설정 파일. 여러 개면 보통 이름 순으로 앞선 것을 기본 네트워크로 사용 |
| stdin | 플러그인에 전달되는 JSON 설정 |
| 환경변수 | `CNI_COMMAND`, `CNI_CONTAINERID`, `CNI_NETNS`, `CNI_IFNAME` 등 |
| stdout | 구성 결과(할당된 IP, 인터페이스, 경로 등) JSON |

**짚고 넘어갈 점:** 플러그인 프로세스는 호출이 끝나면 사라지므로, 상태를 계속 들고 있어야 하는 기능(경로 갱신, 정책 반영 등)은 CNI 실행 파일이 아니라 **별도의 노드 에이전트나 컨트롤러**가 맡는다. 실제 제품이 DaemonSet을 함께 배포하는 이유다.

### 1.3 CNI 동작(verb)

| 동작 | 설명 |
| --- | --- |
| `ADD` | 컨테이너를 네트워크에 연결. 인터페이스 생성, IP 할당, 경로 설정 |
| `DEL` | 연결 해제. 인터페이스 정리와 IP 반환 |
| `CHECK` | 현재 연결 상태가 정상인지 확인 (spec 0.4.0에서 추가) |
| `VERSION` | 플러그인이 지원하는 스펙 버전 응답 |
| `GC` | 런타임이 "살아 있는 목록"을 알려주면 플러그인이 그 외의 누수된 자원(예: 회수되지 않은 IPAM 예약)을 정리 (spec 1.1.0) |
| `STATUS` | 플러그인이 지금 `ADD`를 받을 준비가 됐는지 응답 (spec 1.1.0) |

`GC`와 `STATUS`는 CNI 스펙 1.1.0에서 추가됐다. `STATUS` 이전에는 런타임이 **설정 파일의 존재 여부만으로** 네트워크 준비 상태를 판단해야 했는데, 이제는 플러그인에게 직접 물어볼 수 있다. 설정에 `cniVersions` 목록을 둬서 런타임과 버전을 협상하는 방식도 함께 들어왔다.

### 1.4 conflist와 플러그인 체이닝

하나의 네트워크 설정에는 여러 플러그인을 **순서대로** 나열할 수 있다. 앞 플러그인의 결과가 `prevResult`로 다음 플러그인에 전달된다.

```
{
  "cniVersion": "1.0.0",
  "name": "mynet",
  "plugins": [
    { "type": "bridge",   "bridge": "cni0", "ipam": { "type": "host-local" } },
    { "type": "portmap",  "capabilities": { "portMappings": true } },
    { "type": "bandwidth" }
  ]
}
```

위 예시는 bridge로 인터페이스를 만들고, portmap이 hostPort 규칙을 얹고, bandwidth가 대역폭 제한을 거는 구조다. 기능을 하나의 거대한 플러그인에 몰지 않고 작게 쪼개 조합하는 것이 CNI의 설계 방향이다.

### 1.5 호출 주체 정리 (2주차 복습)

```
sequenceDiagram
    participant K as kubelet
    participant R as Container Runtime
    participant M as CNI Plugin (main)
    participant I as IPAM Plugin

    K->>R: CRI RunPodSandbox
    R->>R: Pod network namespace 준비
    R->>M: CNI_COMMAND=ADD + 설정(JSON)
    M->>I: IPAM 플러그인에 위임 호출
    I-->>M: 할당된 IP / 게이트웨이 / 경로
    M->>M: veth 생성, IP 설정, 경로 구성
    M-->>R: Result JSON
    R-->>K: Pod IP 포함 상태 반환
```

- CNI를 호출하는 주체는 kubelet이 아니라 **컨테이너 런타임**이다.
- `main` 플러그인이 IPAM 플러그인을 **위임 호출**한다. IPAM도 같은 CNI 규약을 따르는 별도 실행 파일이다.

---

## 2. CNI Plugin과 IPAM

### 2.1 플러그인의 세 가지 종류

| 종류 | 역할 | 예시 |
| --- | --- | --- |
| main | 인터페이스를 실제로 만드는 플러그인 | `bridge`, `ptp`, `macvlan`, `ipvlan`, `host-device`, `vlan` |
| ipam | 주소를 할당·반환하는 플러그인 | `host-local`, `dhcp`, `static` |
| meta | 다른 플러그인 위에 기능을 얹는 플러그인 | `portmap`, `bandwidth`, `tuning`, `firewall`, `sbr` |

Flannel, Calico, Cilium 같은 제품은 이 참조 플러그인을 그대로 쓰기도 하고, 자체 main/ipam 구현을 쓰기도 한다.

### 2.2 IPAM 플러그인

| 방식 | 상태를 어디에 두는가 | 특징 |
| --- | --- | --- |
| `host-local` | 노드 로컬 파일 (`/var/lib/cni/networks/<네트워크명>/`) | 가장 단순. API 서버 접근 없이 동작 |
| `dhcp` | 외부 DHCP 서버 | 노드 상주 데몬이 임대(lease) 갱신 필요 |
| `static` | 설정에 직접 명시 | 테스트나 고정 IP 용도 |
| 구현체 자체 IPAM | CRD, 자체 데이터스토어, 클라우드 API | Calico IPAM, Cilium IPAM, AWS VPC CNI 등 |

**짚고 넘어갈 점:** `host-local`은 **자기 노드 안에서의 중복만** 막는다. 클러스터 전체에서 IP가 겹치지 않으려면 노드마다 서로 겹치지 않는 주소 범위를 나눠 주는 상위 단계(2주차의 노드 IPAM 컨트롤러 등)가 필요하다. 반대로 Calico IPAM처럼 자체 데이터스토어를 쓰는 구현은 이 전제를 쓰지 않는다.

### 2.3 IPAM을 볼 때 실제로 갈리는 지점

- **주소 낭비 vs 조회 비용**: 노드마다 큰 블록을 미리 잡아 두면 조회는 단순하지만 주소가 남고, 작은 블록을 필요할 때마다 받아 가면 효율은 좋지만 관리 컴포넌트가 필요하다.
- **고갈 지점이 서로 다름**: 노드별 Pod CIDR인지, Calico IPPool인지, VPC 서브넷과 ENI 슬롯인지에 따라 "IP가 없다"는 증상의 원인 위치가 달라진다.
- **회수(GC)**: Pod가 비정상 종료되거나 `DEL`이 누락되면 예약된 IP가 남을 수 있다. CNI 1.1의 `GC` 동작이 이 문제를 규약 차원에서 다루기 위한 것이다.

---

## 3. Primary CNI Overview

### 3.1 Primary CNI와 보조 네트워크

| 구분 | 설명 |
| --- | --- |
| Primary CNI | Pod의 기본 인터페이스(`eth0`)와 Pod IP를 책임지는 구현체. 클러스터에 하나 |
| 보조 네트워크 | Multus 같은 meta 플러그인이 Pod에 **추가 인터페이스**를 붙이는 방식 |

Multus는 스스로 네트워크를 구현하지 않고, `NetworkAttachmentDefinition` CRD에 정의된 설정을 보고 다른 CNI 플러그인을 대신 호출한다. 따라서 Multus를 쓰더라도 primary CNI는 별도로 필요하다. 주로 NFV, KubeVirt처럼 관리용·데이터용 트래픽을 물리적으로 분리해야 하는 환경에서 쓰인다.

### 3.2 Primary CNI를 비교할 때 보는 축

| 축 | 질문 |
| --- | --- |
| 데이터 경로 | 노드 간 통신을 overlay로 감싸는가, Pod IP 그대로 라우팅하는가 |
| IPAM | 노드별 CIDR을 쓰는가, 자체 풀을 쓰는가, 클라우드 주소를 직접 쓰는가 |
| NetworkPolicy | 지원하는가. L3/L4까지인가, L7·DNS까지인가 |
| Service 처리 | kube-proxy를 그대로 쓰는가, 대체하는가 |
| 부가 기능 | 암호화, 관측성, 멀티클러스터, egress 고정 IP |
| 운영 부담 | 필요한 커널 버전, 디버깅 도구, 팀의 숙련도 |
| 환경 제약 | 관리형 클러스터에서 교체가 가능한가 |

**짚고 넘어갈 점:** 관리형 서비스에서는 선택지가 제한된다. EKS는 AWS VPC CNI, AKS는 Azure CNI 계열, GKE는 Dataplane V2(Cilium 기반)가 기본으로 제공되며, 교체 가능 여부와 지원 범위가 서비스마다 다르다. "어떤 CNI가 좋은가"보다 **"이 환경에서 무엇을 쓸 수 있는가"**가 먼저 결정되는 경우가 많다.

---

## 4. Flannel / Calico / Cilium Overview

### 4.1 Flannel — 연결만 담당하는 최소 구현

| 항목 | 내용 |
| --- | --- |
| 구성 | `flanneld` DaemonSet + Flannel CNI 플러그인 |
| 인터페이스 구성 | 자체 구현 대신 `bridge` + `host-local`에 위임 |
| 주소 할당 | `Node.spec.podCIDR`을 받아 노드 서브넷으로 사용. `/run/flannel/subnet.env`에 기록 |
| 기본 백엔드 | VXLAN (Linux 기본 UDP 8472, 인터페이스 `flannel.1`) |
| 다른 백엔드 | `host-gw` (노드 간 L2 직접 연결 필요, 캡슐화 없음), WireGuard (암호화) |
| NetworkPolicy | 미지원 |

노드 CIDR을 받아 그 안에서 IP를 주고, 다른 노드로는 VXLAN으로 감싸 보내는 것이 전부다. 기능이 적은 만큼 이해하기 쉽고 장애 지점도 적어서 학습이나 소규모 클러스터, k3s 기본 구성에서 자주 보인다.

**흔한 증상:** `flanneld`가 노드 CIDR을 받지 못하면 Pod가 생성되지 않는다. 이때는 Flannel이 아니라 **컨트롤 플레인의 노드 CIDR 할당 설정**을 먼저 확인해야 한다.

### 4.2 Calico — 라우팅과 정책 중심

| 항목 | 내용 |
| --- | --- |
| 구성 | `calico-node`(Felix + BGP 데몬) + 컨트롤러 + Calico CNI + Calico IPAM |
| 데이터 경로 | Pod IP를 유지한 채 L3 라우팅하는 방식이 기본. 필요 시 VXLAN / IP-in-IP 캡슐화 |
| 경로 배포 | BGP로 노드 간, 또는 외부 라우터와 경로 교환 가능 |
| 데이터플레인 선택 | 표준 Linux(iptables 또는 nftables), eBPF, Windows HNS. VPP는 별도 매니페스트 |
| IPAM | Calico IPPool을 노드별 블록(기본 `/26`) 단위로 나눠 사용. 노드 podCIDR에 의존하지 않음 |
| NetworkPolicy | Kubernetes NetworkPolicy + Calico 자체 정책(전역 정책, 명시적 Deny, 규칙 순서 등) |

Felix가 각 노드에서 정책과 경로를 데이터플레인에 반영하는 구조다. eBPF 데이터플레인을 켜면 kube-proxy를 대체하고 출발지 IP 보존이나 DSR 같은 기능을 쓸 수 있다.

**짚고 넘어갈 점:** "Calico = BGP", "Calico = overlay 없음"으로 외우면 안 된다. 캡슐화 기본값은 설치 방식과 플랫폼에 따라 다르므로 **IPPool의 encapsulation 설정을 직접 확인**하는 편이 정확하다.

### 4.3 Cilium — eBPF 기반, 네트워킹 + 보안 + 관측성

| 항목 | 내용 |
| --- | --- |
| 기반 기술 | eBPF. 커널 안에 프로그램을 붙여 패킷을 처리 |
| 정책 모델 | IP가 아니라 레이블에서 파생된 **identity** 기준. L3/L4에 더해 L7(HTTP 메서드·경로), DNS/FQDN 기반 정책 |
| Service | kube-proxy replacement. eBPF 해시 테이블로 부하 분산, 소켓 단계 LB, DSR·Maglev 등 |
| 데이터 경로 | 터널(VXLAN/Geneve) 또는 native routing. BGP 컨트롤 플레인도 내장 |
| IPAM | Cluster Scope(기본), Kubernetes host-scope, Multi-Pool, 클라우드(ENI 등) 등 모드 선택 |
| 부가 기능 | Hubble(플로우 관측), WireGuard/IPsec 암호화, Cluster Mesh, Egress Gateway |
| 전제 조건 | 비교적 최신 리눅스 커널, eBPF 디버깅 도구에 대한 숙련도 |

CNCF Graduated 프로젝트이며, GKE Dataplane V2 등 관리형 서비스의 기반으로도 쓰인다.

**짚고 넘어갈 점:** eBPF는 **커널 안에서** 샌드박스된 프로그램을 실행하는 기술이다. "커널을 우회해 userspace에서 처리한다"는 설명은 정확하지 않다. 또 kube-proxy 프로세스가 없어도 Service 기능 자체는 그대로 존재한다.

### 4.4 한 장 비교

| 항목 | Flannel | Calico | Cilium |
| --- | --- | --- | --- |
| 기본 데이터 경로 | VXLAN overlay | L3 라우팅 + 선택적 overlay | 터널 또는 native routing |
| 주소 할당 | `host-local` + 노드 podCIDR | Calico IPAM (IPPool/블록) | Cilium IPAM (모드 선택) |
| NetworkPolicy | 미지원 | K8s 정책 + 자체 확장 정책 | K8s 정책 + L7/FQDN 정책 |
| Service 처리 | kube-proxy | kube-proxy 또는 eBPF 모드로 대체 | eBPF로 대체 가능 |
| 관측성 | 별도 | 메트릭 중심 | Hubble 내장 |
| 운영 난이도 | 낮음 | 중간 | 상대적으로 높음 |
| 자주 쓰이는 상황 | 학습, 소규모, k3s 기본 | 정책 요구가 큰 환경, 온프렘·BGP 연동 | 대규모, 관측성·보안 요구가 큰 환경 |

기능이 많은 쪽이 항상 정답은 아니다. 커널 버전, 팀의 디버깅 역량, 관리형 서비스의 제약이 실제 선택을 결정하는 경우가 많다.

### 4.5 트러블슈팅 관점에서의 순서

CNI 설정이나 플러그인이 없으면 Pod는 대개 `ContainerCreating`에서 멈춘다. sandbox 생성 단계에서 네트워크를 붙이지 못하기 때문이다. 확인 순서는 대략 이렇게 잡을 수 있다.

1. `/etc/cni/net.d`에 설정 파일이 있는가, 어떤 것이 선택되는가
2. `/opt/cni/bin`에 해당 플러그인 실행 파일이 있는가
3. CNI 에이전트 DaemonSet Pod가 정상인가, 로그에 무엇이 찍히는가
4. 주소가 부족한가 — **해당 구현이 쓰는 IPAM 자원**(노드 CIDR / IPPool / VPC 서브넷)을 확인
5. 연결은 되는데 통신이 안 되는가 — 경로, 캡슐화 설정, MTU 확인

`hostNetwork: true`인 Pod는 노드 네트워크를 그대로 쓰므로 위 증상에서 예외다. 이 점을 이용해 "CNI 문제인지 아닌지"를 구분할 수 있다.

---

## 5. Cloud Native Networking Landscape

지금 생태계의 흐름을 정리하면 대략 세 가지다.

**① 인터페이스는 표준, 구현은 경쟁**
CNI가 연결 지점을 표준화한 덕분에 구현체는 자유롭게 경쟁할 수 있게 됐다. 다만 CNI가 다루지 않는 영역(정책, Service, 멀티클러스터)은 구현체마다 방식이 다르고, 그래서 Kubernetes NetworkPolicy나 Gateway API처럼 **CNI 바깥에서 별도로 표준화**되는 API들이 계속 늘어나고 있다.

**② iptables에서 eBPF로**
Service와 정책 규칙이 많아질수록 iptables 체인 방식은 부담이 커진다. kube-proxy의 nftables 모드, Calico의 eBPF 데이터플레인, Cilium의 kube-proxy replacement 모두 같은 문제의식에서 나온 흐름이다.

**③ 네트워킹·보안·관측성의 통합**
예전에는 CNI가 연결만 담당하고 정책은 별도 도구, 관측은 또 다른 도구였다. 지금은 하나의 구현체가 연결·정책·암호화·플로우 관측까지 함께 제공하는 방향으로 가고 있다. 사이드카 없는 서비스 메시 논의도 같은 맥락이다.

### 이번 주 정리

- CNI는 **"연결해 달라"는 요청 방식**만 표준화했고, 방법은 구현체가 정한다.
- 플러그인은 데몬이 아니라 실행 파일이며, 상태 유지가 필요한 일은 별도 에이전트가 맡는다.
- IPAM은 CNI 안의 또 하나의 위임 지점이고, **어떤 IPAM을 쓰는지가 주소 고갈 문제의 성격을 결정한다.**
- Flannel은 연결만, Calico는 라우팅과 정책, Cilium은 eBPF로 Service·정책·관측까지 가져간다. 기능 범위가 곧 운영 부담이기도 하다.

---

## 참고 자료

- [CNI — Specification](https://github.com/containernetworking/cni/blob/main/SPEC.md)
- [CNI — Spec versions (upgrade notes)](https://www.cni.dev/docs/spec-upgrades/)
- [CNI — Plugins](https://www.cni.dev/plugins/current/)
- [Kubernetes — Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
- [Kubernetes — Container Runtime Interface](https://kubernetes.io/docs/concepts/containers/cri/)
- [Flannel — Backends](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md)
- [K3s — Network Options (Flannel backends)](https://docs.k3s.io/installation/network-options/)
- [Calico — Get started with IP address management](https://docs.tigera.io/calico/latest/networking/ipam/get-started-ip-addresses)
- [Calico — Install in eBPF mode](https://docs.tigera.io/calico/latest/operations/ebpf/install)
- [Cilium — Documentation](https://docs.cilium.io/)
- [Multus CNI](https://github.com/k8snetworkplumbingwg/multus-cni)
