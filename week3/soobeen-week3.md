# Week 3. CNI Overview & Cloud Native Networking Landscape

## 1. CNI가 해결하려는 문제 (등장 배경)

- CNI 이전: Docker는 **CNM(Container Network Model, libnetwork)**이라는 자체 네트워크 모델을 씀 → **Docker 데몬에 강하게 종속**된 구조라 다른 컨테이너 런타임(containerd, CRI-O)이 그대로 가져다 쓰기 어려움
- **CNI**: "실행 파일 하나 + 환경변수 + stdin/stdout JSON"이라는 최소한의 계약만 정의 → 어떤 런타임이든 이 계약만 지키면 같은 플러그인 바이너리를 공유해서 쓸 수 있도록!
- 단점 사례(Red Hat의 Podman)
  - **Podman**은 원래 CNI를 썼지만, 지금은 자체 개발한 **netavark + aardvark-dns**로 완전히 갈아탐
  - 이유: Podman은 **데몬리스**(매번 새 프로세스가 직접 컨테이너를 만듦) 구조라, 컨테이너 하나 뜰 때마다 CNI 플러그인 바이너리를 fork+exec 하는 방식이 **느리고**, **rootless**(root 권한 없이 컨테이너 실행) 환경에서 상태 일관성을 맞추기 어려웠음
  - netavark는 "원하는 상태(desired state)를 커널 상태와 비교해서 **차이만 수렴**시키는" 방식으로 바꿔서 이 문제를 해결 — CNI의 "표준화"라는 장점과 "매번 프로세스 실행"이라는 한계를 동시에 보여주는 좋은 대조 사례
- 즉 CNI는 **만능이 아니라 "런타임 독립성"이라는 한 가지 문제를 표준 인터페이스로 푼 것**이고, 그 인터페이스 방식 자체의 트레이드오프(느림, 상태 관리 부담)는 여전히 남아있음

## 2. CNI가 표준 인터페이스로서 제공하는 것

| 구성 | 내용 |
|---|---|
| 환경변수 | `CNI_COMMAND`(`ADD`\|`DEL`\|`CHECK`\|`GC`\|`VERSION`\|`STATUS`), `CNI_CONTAINERID`, `CNI_NETNS`(어느 네임스페이스에), `CNI_IFNAME`(어떤 인터페이스 이름으로), `CNI_PATH` |
| stdin | 네트워크 설정 JSON ("어떤 설정으로") |
| stdout | 결과 JSON (할당된 IP, 라우트 등) — 성공 시 exit code 0, 실패 시 에러 JSON |

- `ADD`로 인터페이스를 만들고, `DEL`로 정리하고, `CHECK`로 상태를 확인
- **표준 인터페이스가 제공하는 실질적 이득**
  - **런타임 독립성**: containerd, CRI-O이 `/opt/cni/bin`의 **동일한 플러그인 바이너리**를 그대로 공유해서 씀
  - **책임 분리**: 런타임은 "이 netns에 네트워크 만들어줘"라고 요청만 하고, 실제 veth/라우트/IP 설정 방법은 전혀 몰라도 됨
  - **플러그인 갈아끼우기**: 같은 계약만 지키면 Flannel을 Calico로 바꿔도 kubelet·CRI 쪽 코드는 손댈 필요 없음

### 스펙이 정의하는 것 vs 정의하지 않는 것

| 정의하는 것 | 정의하지 않는 것 |
|---|---|
| 실행 파일 호출 규약(환경변수+stdin/stdout JSON) | **네트워크 구현 방식**(veth를 쓰든 BGP를 켜든 자유) |
| 설정 파일 포맷(`.conflist`) | **노드 간 연결** |
| 결과 타입(`interfaces`, `ips`, `routes`, `dns`) | IP 대역 관리 정책 |
| 오퍼레이션(`ADD`/`DEL`/`CHECK`/`GC`/`STATUS`/`VERSION`) | **NetworkPolicy** |

> 📌 **NetworkPolicy는 스펙 밖**이라는 게 의외로 중요한 포인트. Kubernetes는 `NetworkPolicy`라는 **리소스를 정의만** 하고, 그걸 실제로 집행하는 건 순전히 CNI 구현체의 **자발적 기능**. 그래서 스펙만 준수하는 플러그인(Flannel 단독 구성 등)은 NetworkPolicy를 만들어도 **에러 없이 조용히 무시**함

## 3. CNI Plugin / IPAM

### 3종류로 분리

| 타입 | 역할 | 예 |
|---|---|---|
| **main** | 인터페이스 생성 (veth 등) — conflist에서 **반드시 첫 번째, 단 하나만** | `bridge`, `calico`, `cilium-cni` |
| **ipam** | 주소 할당만 전담. 체인의 원소가 아니라 **main 설정 안에 중첩**되어 main이 실행 중 직접 호출 | `host-local`, `calico-ipam`, `dhcp` |
| **meta** | 완성된 결과에 기능을 덧붙임 | `portmap`, `bandwidth`, `tuning`, `firewall` |

- `conflist` 설정에서 `ipam.type`으로 지정하면, main 플러그인이 그 IPAM 바이너리를 **별도로 다시 호출**함
  ```json
  { "type": "calico", "ipam": { "type": "calico-ipam" } }
  ```
- **capabilities / runtimeConfig**: meta 플러그인(`portmap` 등)이 Pod 스펙의 동적인 값(예: `hostPort`)을 받으려면, conflist에 `"capabilities": {"portMappings": true}`처럼 **"이 값을 런타임에서 받을 수 있다"고 선언**해둬야 함. 그러면 런타임이 실행 시점에 `runtimeConfig` 키로 실제 값을 stdin JSON에 주입함(선언 안 하면 주입X)
  ```
  Pod spec hostPort: 8080
    → containerd가 portmap의 portMappings capability 선언 확인
    → stdin JSON에 "runtimeConfig": {"portMappings": [...]} 추가
    → portmap이 iptables DNAT 규칙 생성
  ```

### IPAM이 왜 필요한가
- CNI 플러그인은 `ADD` 한 번 실행되고 **바로 종료되는 프로세스** → Stateless
- 그런데 "이 IP는 이미 다른 Pod가 쓰고 있다"는 사실은 **다음 호출 때 반드시 알아야** 함 → 상태를 어딘가에 반드시 남겨야 함
- 이 "주소 할당 상태 관리"라는 책임을 메인 플러그인에서 떼어내 별도 모듈(IPAM)로 분리해두면:
  - 네트워크 연결 방식과 주소를 어디서 가져올지를 **독립적으로 갈아끼울 수 있음**

### conflist 체이닝
- CNI는 여러 플러그인을 **순서대로** 실행 가능 -> 첫 번째 플러그인만 인터페이스를 만들고, 나머지는 앞 단계 결과(`prevResult`)를 이어받아 기능을 덧붙임
  ```json
  "plugins": [
    { "type": "calico", "ipam": {"type": "calico-ipam"} },
    { "type": "portmap", "capabilities": {"portMappings": true} },
    { "type": "bandwidth", "capabilities": {"bandwidth": true} }
  ]
  ```

## 4. Primary CNI

### 먼저, "Primary CNI"란 정확히 뭔가
`Primary CNI`는 스펙 용어가 아니라 실무 용어. 아래 조건을 다 채워야 "Primary"라고 부름:
1. Pod 기본 인터페이스(`eth0`) 생성
2. **클러스터 전역 Pod-to-Pod 도달성 확보**
3. 통상 NetworkPolicy 집행

이 도달성 확보는 **① 노드 안**(CNI 스펙이 정의하는 범위, `kubelet→containerd→플러그인 실행`)만으로는 안 되고, **② 노드 간**(스펙이 정의 안 하는 범위, 구현체가 알아서 담당)이 반드시 필요함

| 구분 | 방식 | 대표 |
|---|---|---|
| 오버레이 (캡슐화) | Pod 패킷을 노드 간 터널에 실음, 물리망은 Pod IP 몰라도 됨 | Flannel VXLAN, Calico IPIP/VXLAN, **Cilium VXLAN/Geneve** |
| 언더레이 (라우팅) | 물리 라우터에 Pod CIDR 경로를 직접 광고 | Flannel host-gw, Calico BGP, **Cilium native routing** |
| 클라우드 네이티브 IPAM | Pod에 VPC 실제 주소를 직접 할당 | AWS VPC CNI, Azure CNI |


### 4-1. Flannel — 연결만 확실히 하는 최소 구현
- 설계 철학이 명확함: **최소한만 직접 구현하고, 노드 안(①) 데이터패스는 표준 `bridge` 플러그인에 그대로 위임**
- `flanneld`(DaemonSet)와 `flannel` CNI 플러그인은 **직접 호출 관계가 아니라 파일 하나로 느슨하게 연결**됨
  1. `flanneld`가 API 서버에서 자기 Node의 `podCIDR` 확인 → `/run/flannel/subnet.env`에 기록 (`FLANNEL_SUBNET=10.244.0.1/24`, `FLANNEL_MTU=1450` 등)
  2. `flannel` CNI 플러그인이 `ADD` 호출 시 이 파일을 읽어서 `{"type":"bridge","bridge":"cni0","ipam":{"type":"host-local",...}}` 설정을 만들고 **`bridge` 플러그인을 다시 호출(위임)**
  3. 즉 노드 안 데이터패스는 **reference bridge 플러그인 그대로**임 — Flannel 고유 영역은 오직 **②(노드 간, VXLAN 터널/FDB 관리)**뿐
- 전달 모드는 기본 **VXLAN** 또는 **host-gw**(캡슐화 없이 상대 Node IP를 게이트웨이로 라우팅 테이블에 직접 기록, Node끼리 L2로 붙어있어야 함)
- NetworkPolicy 자체 구현이 없어서, 정책까지 필요하면 **Canal**(연결=Flannel + 정책=Calico 정책 엔진만 결합)로 보완하는 게 일반적

### 4-2. Calico — 각 노드를 라우터로 만든다
- `calico-node`(DaemonSet, 컨테이너 하나에 프로세스 여럿) 안에 역할이 나뉨: **BIRD**(BGP 데몬), **confd**(데이터스토어를 감시해서 BIRD 설정을 렌더링), **Felix**(라우트·iptables/nftables/eBPF를 실제 커널에 프로그래밍), **Typha**(대규모 클러스터에서 API Server watch 부하를 흡수, 보통 Node 50대 이상에서 사용)
- **브리지가 아예 없는 `/32` 라우팅 모델**: Pod의 veth가 Node에 직접 붙고(브리지 경유 X), Node가 각 Pod를 `/32`(호스트 라우트) 단위로 자기 라우팅 테이블에 등록해서 전달
- 축이 두 개로 나뉨: **노드 간(②)** = BGP(오버레이 없음) / IPIP / VXLAN / CrossSubnet 중 선택, **노드 안(①)** = iptables(기본) / nftables / **eBPF**(선택 시 kube-proxy 대체 가능)
- IPAM이 CRD(`IPPool → IPAMBlock → BlockAffinity`) 기반이라 Node가 죽어도 할당 상태가 유지·복구 가능
- **NetworkPolicy**: K8s 표준은 물론 `GlobalNetworkPolicy`, 정책 순서(tier), DNS 정책까지 자체 확장 제공

### 4-3. Cilium — 커널 패킷 처리 자체를 eBPF로 대체
- `cilium-agent`(DaemonSet)가 Service·NetworkPolicy 정보를 **BPF 프로그램/맵으로 컴파일해서 커널의 훅(tc/tcx/XDP/cgroup)에 직접 부착**
- **kube-proxy replacement**: `cilium_lb4_services_v2` 같은 BPF 맵 조회로 ClusterIP→Pod IP 변환을 대체
- **Identity 기반 정책**: Pod label 조합을 숫자 identity로 미리 컴파일해두고, 정책을 IP가 아니라 identity 간 규칙으로 기술 → Pod가 재생성돼 IP가 바뀌어도 정책은 그대로 유효
- **Hubble**: agent가 기록한 BPF 이벤트를 `hubble-relay`가 모아서 UI로 스트리밍
- 트레이드오프: 기존 커널 도구(`iptables -L`, `conntrack -L`)로 상태를 못 보고 `cilium monitor`, `cilium bpf ct list` 같은 전용 도구가 필요

## 5. 선택 기준 — "어떤 상황에서 어떤 걸 써야 하는가"

1. **물리망에 Pod CIDR 경로를 등록할 수 있는가?**
   - 아니오(퍼블릭 클라우드 VPC라 라우팅 테이블을 내 마음대로 못 건드림) → **오버레이**(Flannel VXLAN, Calico VXLAN/IPIP, Cilium) 또는 클라우드 네이티브 CNI(VPC CNI)만 선택 가능
   - 예(온프레미스라 라우터 설정 가능) → 모든 Node가 같은 L2 세그먼트인지에 따라 **host-gw/no-overlay**(같은 L2) 또는 **BGP**(다른 세그먼트, 라우터가 BGP 피어링 가능하면)

2. **요구사항별 선택**

| 요구사항 | 선택 |
|---|---|
| 단순함, 학습·소규모 | Flannel VXLAN |
| NetworkPolicy 필요 | Calico, Cilium (Flannel 단독 불가) |
| 물리망 BGP 통합 | Calico |
| L7 정책 / 관측성 / kube-proxy 제거 | Cilium |
| 클라우드 VPC 통합 | 해당 클라우드 CNI |

3. **운영 비용도 같이 고려**: 학습 곡선은 Flannel(낮음) < Calico(중간) < Cilium(높음) 순. 트러블슈팅 난이도도 같은 순서
4. **CNI 교체는 무중단이 거의 불가능** — Pod IP 대역과 데이터패스 자체가 바뀌는 거라, 보통 신규 클러스터를 만들어 워크로드를 옮기는 방식으로 처리함 → **그래서 초기 선택의 비중이 큼**

## 6. CNI 계약이 깨지면? — Node NotReady 메커니즘

- CNI 설정 로드에 실패하면: containerd가 `lastCNILoadStatus`를 기록 → kubelet이 이를 `RuntimeStatus.conditions[NetworkReady] = false`로 받아들임 → **Node 전체가 `NotReady`(`NetworkPluginNotReady`)**
- "네트워크 플러그인 하나 때문에 Node 전체가 막히는 게 과한가?" → CNI가 없으면 그 Node에 뜨는 **어떤 Pod도 IP를 못 받음** → 스케줄해봐야 전부 실패하므로, 아예 배치 대상에서 빼는 게 합리적
- 그래서 새 클러스터에서 CNI를 설치하기 전까지 Node가 계속 `NotReady`인 것도 **정상 동작**

## 정리: 이번 주 3가지 질문에 답하기

**Q1. CNI가 표준 인터페이스로서 제공하는 것은?**
"실행 파일 + 환경변수 + stdin/stdout JSON"이라는 최소 계약. 이 계약 덕분에 컨테이너 런타임(containerd, CRI-O 등)이 특정 벤더에 종속되지 않고 동일한 CNI 플러그인 바이너리를 공유해서 쓸 수 있음. Docker의 CNM이 데몬 종속적이라 다른 런타임이 채택하기 어려웠던 것과 대조됨.

**Q2. IPAM이 왜 필요한가?**
CNI 플러그인 프로세스는 실행되고 바로 종료되어 상태를 못 들고 있는데, "이 IP는 이미 사용 중"이라는 정보는 다음 호출 때 반드시 필요함. 이 상태 관리 책임을 메인 네트워크 연결 플러그인에서 분리해 별도 모듈로 만든 것이 IPAM

**Q3. 각 CNI의 기본 아키텍처 차이는?**
"노드 간(②) 연결을 어떻게 하는가"와 "노드 안(①) 패킷 처리를 무엇으로 하는가"는 서로 다른 축임. Flannel은 ②를 오버레이(VXLAN)로 풀고 ①은 표준 bridge 플러그인에 위임. Calico는 ②를 BGP 라우팅(브리지 없는 `/32` 경로)으로 풀고 ①은 iptables가 기본. Cilium은 ②를 오버레이/라우팅 둘 다 지원하되 ①을 iptables 대신 eBPF로 처리해서 kube-proxy까지 흡수. 

## 질문

1. Podman의 netavark처럼 "상태를 커널과 비교해서 수렴시키는" 방식이 멱등성이나 선언형이라 CNI의 exec-per-call 방식보다 나아 보이는데, 왜 Kubernetes 진영은 CNI 표준을 netavark 같은 방식으로 바꾸지 않고 계속 유지하는지?(벤더 종속 안 돼서?)
2. Calico의 `/32` 라우팅 모델에서, Node의 라우팅 테이블에 Pod 하나당 라우트가 하나씩 쌓이는 거라면 Pod가 수백 개인 대규모 클러스터에서는 라우팅 테이블 크기 자체가 성능 이슈가 되지는 않는지? (BGP 세션 수는 Route Reflector로 줄인다고 알고 있는데 라우트 개수 자체는 어떻게 관리하는지)

## 참고 자료
- [CNI 공식 스펙](https://github.com/containernetworking/cni)
- [Flannel 공식 문서](https://github.com/flannel-io/flannel)
- [Calico 공식 문서](https://docs.tigera.io/calico/latest/about/)
- [Cilium 공식 문서](https://docs.cilium.io/)