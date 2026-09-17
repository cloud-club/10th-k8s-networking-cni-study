## 1. CNI 개요

### CNI가 해결하는 문제

- 런타임마다 네트워크 연동 방식이 달라지는 문제
- 런타임과 네트워크 구현 사이에 **공통 호출 규약**을 두어 결합도 감소
- 런타임은 규약대로 호출만, 네트워크 솔루션은 규약에 맞는 플러그인만 제공
- Kubernetes 전용이 아닌 컨테이너 일반을 위한 CNCF 규약

### 용어: 런타임

- kubelet의 CRI 요청을 처리하는 **containerd, CRI-O**
- CNI 플러그인을 직접 실행하는 주체
- runc 같은 저수준 런타임은 CNI 호출과 무관

### CNI가 다루는 범위

- 컨테이너를 네트워크에 **연결·해제**하는 시점의 작업
- 노드 간 경로, NetworkPolicy, Service 처리는 **규약 범위 밖**
- 해당 기능은 각 솔루션의 에이전트·컨트롤러가 담당

```bash
[CNI 없는 구조]

런타임 A ── 전용 연동 ── 네트워크 X
런타임 A ── 전용 연동 ── 네트워크 Y
런타임 B ── 전용 연동 ── 네트워크 X
런타임 B ── 전용 연동 ── 네트워크 Y

[CNI 있는 구조]

런타임 A ─┐                ┌─ 네트워크 X
          ├── CNI 규약 ──┤
런타임 B ─┘                └─ 네트워크 Y
```

---

## 2. CNI가 표준 인터페이스로서 제공하는 것

### 호출 방식

```
런타임
   │ 환경 변수: CNI_COMMAND, CNI_NETNS, CNI_IFNAME ...
   │ stdin    : 네트워크 설정 JSON
   ▼
CNI 플러그인 실행 파일
   │ stdout   : 결과 JSON (인터페이스, IP, 라우트)
   ▼
런타임
```

- 플러그인은 **실행 파일**, 호출 후 종료
- 설정 파일: `/etc/cni/net.d`, 실행 파일: `/opt/cni/bin`

### 주요 명령

| 명령 | 역할 |
| --- | --- |
| `ADD` | 네트워크 연결, 결과 반환 |
| `DEL` | 네트워크 해제, 리소스 정리 |
| `CHECK` | 연결 상태 확인 |
| `VERSION` | 지원 스펙 버전 보고 |

### 플러그인 체이닝

 CNI 플러그인 여러 개를 **순서대로 이어서 실행**해 하나의 네트워크 연결을 완성하는 방식

- 설정의 `plugins` 배열 순서대로 실행
- 앞 플러그인 결과를 `prevResult`로 전달
- 예: `bridge`(연결) → `portmap`(포트 매핑)

### 노드 상태와의 관계

- 유효한 CNI 설정이 없으면 노드가 `NotReady`
- 클러스터 구축 직후 CNI 설치 전 `NotReady`인 이유

---

## 3. CNI Plugin / IPAM

### 플러그인 분류

| 분류 | 역할 | 예시 |
| --- | --- | --- |
| Interface | 인터페이스 생성·연결 | `bridge`, `ptp`, `macvlan` |
| IPAM | IP 할당·회수 | `host-local`, `dhcp` |
| Meta | 부가 기능 | `portmap`, `bandwidth` |

### IPAM이 필요한 이유

- **중복 방지:** 같은 IP를 가진 Pod는 통신 불가
- **회수·재사용:** 반환하지 않으면 주소 고갈
- **상태 보존:** 플러그인은 실행 후 종료되므로 할당 기록을 별도 저장
- **동시성:** 여러 Pod 동시 생성 시 충돌 방지

### CNI별 IPAM

| CNI | 기본 방식 | 특징 |
| --- | --- | --- |
| Flannel | host-local | 노드별 고정 서브넷 (`/24`) |
| Calico | calico-ipam | IPPool을 블록(`/26`) 단위로 노드에 동적 배정 |
| Cilium | cluster-pool | operator가 노드별 CIDR 배정 |
- IPAM 방식에 따라 주소 체계와 라우팅 구성이 달라짐

---

## 4. Primary CNI Overview

- **Primary CNI:** Pod의 기본 네트워크(`eth0`)와 Pod 간 통신 제공
- 일반적으로 클러스터당 하나 사용

| 분류 | 예시 |
| --- | --- |
| 단순 Overlay | Flannel |
| 라우팅·정책 중심 | Calico |
| eBPF 중심 | Cilium |
| 클라우드 연동 | AWS VPC CNI, Azure CNI |

### 비교 관점

- 노드 간 전달: Overlay / Native Routing / BGP
- 데이터플레인: bridge·iptables / eBPF
- NetworkPolicy 지원 여부
- Service 처리: kube-proxy 사용 / 대체

---

## 5. Flannel

- **노드 간 L3 연결**에 집중한 단순한 구현
- NetworkPolicy 미지원

| 구성요소 | 역할 |
| --- | --- |
| flanneld | 노드 서브넷 확인, 백엔드·경로 구성 |
| flannel CNI | bridge + host-local에 위임 |

```
Pod ─ veth ─ cni0 (bridge) ─ flannel.1 (VXLAN) ═ 노드 간 터널
```

- 백엔드: `vxlan`(기본), `host-gw`(캡슐화 없음, L2 인접 필요)

---

## 6. Calico

- **L3 라우팅 + NetworkPolicy** 강점
- bridge 없이 호스트 라우팅 테이블로 전달

| 구성요소 | 역할 |
| --- | --- |
| Felix | 라우트·정책 규칙 프로그래밍 |
| BIRD | BGP로 노드 간 Pod 경로 교환 |
| calico-ipam | 블록 기반 IP 할당 |

```
Pod ─ veth (caliXXX) ─ 호스트 라우트 (/32) ─ BGP 경로 ─ 다른 노드
```

- 노드 간 전달: BGP(캡슐화 없음), IPIP, VXLAN
- 데이터플레인: iptables, nftables, eBPF

---

## 7. Cilium

- **eBPF 기반** 연결·Service·정책·관측 통합
- IP가 아닌 **라벨 기반 Security Identity**로 정책 집행
- L7 정책, kube-proxy 대체, Hubble 관측 제공

| 구성요소 | 역할 |
| --- | --- |
| cilium-agent | eBPF 프로그램·맵 관리 |
| cilium-operator | IPAM CIDR 배정 등 클러스터 작업 |
| Hubble | 네트워크 흐름 관측 |

```
Pod ─ veth (lxcXXX, eBPF) ─ 정책·Service 처리 ─ VXLAN/Geneve 또는 Native
```

---

## 8. 비교 정리

| 항목 | Flannel | Calico | Cilium |
| --- | --- | --- | --- |
| 핵심 방향 | 단순 연결 | 라우팅 + 정책 | eBPF 통합 |
| 노드 내 연결 | bridge | veth + 라우트 | veth + eBPF |
| 노드 간 전달 | VXLAN, host-gw | BGP, IPIP, VXLAN | VXLAN, Native |
| 기본 IPAM | 노드 서브넷 | 블록 | cluster-pool |
| NetworkPolicy | 미지원 | 지원 | 지원 (L7 포함) |
| kube-proxy 대체 | X | eBPF 모드 | O |
| 운영 복잡도 | 낮음 | 중간 | 높음 |

### 핵심 구분

- CNI는 **연결·해제 시점의 표준 호출 규약**
- IPAM은 **상태를 가진 주소 할당** 담당
- 세 솔루션 모두 CNI 플러그인을 제공하지만 **플러그인 뒤의 아키텍처가 다름**
- NetworkPolicy 리소스 생성과 실제 집행은 별개
