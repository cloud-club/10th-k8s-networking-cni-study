## 1. eBPF 동작 원리

<img width="1360" height="712" alt="image" src="https://github.com/user-attachments/assets/015d6699-4650-405a-b573-5d3701694e0a" />


- 유저 공간 도구는 `bpf()` 시스템 콜로만 커널에 접근함
- 커널 안에서 Verifier·JIT가 검증·번역, Map이 상태 저장, Helper가 커널 기능 제공
- 검증된 프로그램은 hook(XDP·tc, kprobe, cgroup 등)에 붙어 실행됨

### 동작 과정

<img width="1360" height="1124" alt="image" src="https://github.com/user-attachments/assets/b98c934f-c22b-4f2d-a353-ce380032da10" />


- C로 작성한 코드를 clang/LLVM으로 BPF 바이트코드로 컴파일함
- 로더가 `bpf()` 시스템 콜로 Map을 만들고 프로그램을 커널에 넘김
- Verifier가 안전성을 검사하고, 통과하면 JIT이 기계어로 바꿈 (실패하면 로드 자체가 안 됨)
- 지정한 hook에 붙인 뒤, 이벤트가 hook을 지날 때마다 실행됨
- Cilium에서는 cilium-agent가 이 과정을 대신 해 줌 (컴파일 → 로드 → attach)

### Verifier

- 커널 모듈과 eBPF의 가장 큰 차이
    - 커널 모듈: 버그가 있으면 커널이 같이 죽음
    - eBPF: 위험한 프로그램은 로드 단계에서 거부됨
- 검사하는 것
    - 반드시 끝나는지 (무한 루프 X, 종료가 보장되는 bounded loop만 O)
    - 초기화 안 된 값 사용, 패킷·메모리 범위 밖 접근이 없는지
    - Map 조회 결과를 NULL 검사 후 쓰는지
    - 크기·복잡도가 한도(처리 명령 100만 개) 안인지
- 모든 분기를 따라가며 레지스터 타입과 값 범위를 추적하는 방식
- 주의: 커널을 망가뜨리지 않는지만 보는 **safety** 도구이고, 프로그램 의도까지 보는 security 도구는 아님

### JIT

- 검증된 바이트코드를 CPU 기계어(x86_64, arm64 등)로 번역함
- 그래서 커널 원래 코드와 비슷한 속도로 동작
- `net.core.bpf_jit_enable` 로 활성화 여부 확인 (1이면 켜짐)

### Hook

<img width="1360" height="1120" alt="image" src="https://github.com/user-attachments/assets/778d0b83-bc8c-473b-8268-ef316f3fc891" />


- eBPF는 이벤트 기반 → 어디에 붙느냐 = 언제 실행되느냐
- 보라색이 BPF가 붙는 지점, 회색이 커널 기본 처리

| Hook | 실행 시점 | Cilium에서 쓰는 곳 |
| --- | --- | --- |
| XDP | NIC 드라이버가 패킷을 받자마자 | prefilter, NodePort 가속 |
| tc ingress / egress (tcx) | sk_buff 생성 후, L3 처리 전 | L3/L4 정책, LB, redirect |
| cgroup (connect 등) | 소켓 시스템 콜 시점 | socket-LB (ClusterIP → backend 변환) |
- Pod 트래픽은 대부분 tc에서 처리됨(Traffic Control)
    - Pod veth의 호스트 쪽(lxc) tc ingress에 `cil_from_container` 를 붙임
    - Pod가 *보낸* 패킷이 호스트 쪽에서는 *ingress* 로 들어오기 때문
- eBPF는 아무 커널 함수나 못 부름 → 커널이 열어 둔 helper 함수만 사용 (Map 조회, 패킷 수정 등)

### Tail call

<img width="1360" height="700" alt="image" src="https://github.com/user-attachments/assets/b8624227-7fa4-44de-98f3-a77206303a34" />


- Tail call: 다른 BPF 프로그램으로 점프하고 돌아오지 않음
    - 점프 대상은 따로 검증됨 → Cilium은 큰 datapath를 tail call로 쪼개서 Verifier 한도를 피함
- BPF-to-BPF call: 일반 함수처럼 호출 후 복귀

---

## 2. BPF Map & Cilium Datapath

### BPF Map

<img width="1360" height="840" alt="image" src="https://github.com/user-attachments/assets/0b44ff1e-a2ad-4fb3-b3b0-574c3f8d4923" />


- eBPF 프로그램은 패킷 하나 처리하고 끝나서 상태를 기억 못 함 → 유지해야 할 데이터는 Map에 둠
- 특징
    - BPF 프로그램끼리, 유저 공간 프로그램과도 공유 가능
    - 생성할 때 최대 크기가 정해짐 → 꽉 차면 삽입 실패
    - 종류: Hash, Array, LRU, LPM, Ring Buffer 등
- Cilium에서의 역할: control plane과 data plane을 이어 줌
    - cilium-agent: K8s 상태 계산 → identity, 정책, Service 정보를 Map에 씀
    - BPF 프로그램: 패킷마다 Map을 조회
    - 정책이 바뀌어도 프로그램은 그대로, Map 내용만 바뀜
- iptables와 비교: 규칙을 위에서부터 훑는 대신 key로 바로 찾음 → 규칙이 늘어도 조회 비용이 거의 안 늘어남
- Map 위치: `/sys/fs/bpf/tc/globals/` 에 pin됨 → agent가 재시작돼도 Map이 남아 트래픽 처리 유지

### Identity & ipcache

<img width="1360" height="660" alt="image" src="https://github.com/user-attachments/assets/043700b8-6645-4916-8be6-845bcfd4ca52" />


- 같은 label의 Pod는 같은 identity(숫자)를 공유
- 정책은 IP가 아니라 identity 기준으로 저장됨
- 패킷에는 IP만 있으니 ipcache에서 IP → identity를 찾음 (정책 판단의 시작점)
- Pod가 늘어나면 ipcache 항목만 추가되고 정책 Map은 그대로

### Cilium이 쓰는 주요 Map

| Map | 이름 (v1.20.2) | 기본 한도 | 들어 있는 것 |
| --- | --- | --- | --- |
| IP cache | `cilium_ipcache_v2` | 512k | IP/CIDR → identity, 목적지 노드 |
| Policy | `cilium_policy_v3_<EP>` | endpoint당 16k | 허용할 identity + 포트 + 프로토콜 + 방향 |
| Service LB | `cilium_lb4_services_v2` | 64k | Service VIP와 backend 슬롯 |
| Backends | `cilium_lb4_backends_v3` | 64k | 실제 backend Pod IP:port |
| Conntrack | `cilium_ct4_global` 등 | TCP 512k / 그 외 256k | 연결 상태 |
- 이름 뒤 버전 접미사는 Cilium 버전마다 다름
- Policy map: 항목이 있으면 허용, 없으면 drop. CIDR를 잘게 쪼갠 정책 등은 16k를 채울 수 있음
- CT map: 응답 패킷(`CT_REPLY`)은 정책 검사를 건너뜀 (stateful)

### Map을 참조해서 패킷을 처리하는 과정

<img width="1360" height="952" alt="image" src="https://github.com/user-attachments/assets/f528fac9-2bd2-4b30-b8b7-ff4eef484aba" />


Pod A(node1)가 Service를 거쳐 Pod B(node2)로 가는 경우 (overlay 모드)

1. LB Map: Service IP → backend Pod IP로 변환
2. CT: 새 연결인지 확인
3. ipcache: 목적지 identity와 목적지 노드 확인
4. egress 정책: Pod A가 내보내도 되는지 판단
5. VXLAN으로 node2에 전달 (VNI에 출발지 identity를 실음)
6. node2: decap → `cilium_lxc` 로 목적지 Pod 확인 → 정책 프로그램으로 tail call
7. ingress 정책: Pod B가 받아도 되는지 판단 → 통과하면 전달
- 정책 검사는 송신 쪽 egress, 수신 쪽 ingress로 **두 번** 일어남
- 4번이나 7번에서 막히면 Hubble에 DROPPED로 남음
- socket-LB가 켜져 있으면 1번은 `connect()` 시점에 먼저 처리됨

---

## 3. Hubble & Troubleshooting

### 구성

<img width="1360" height="780" alt="image" src="https://github.com/user-attachments/assets/d5909150-cfeb-448d-96b3-9e30b1925025" />


- Hubble server: 노드마다 cilium-agent 안에 내장, 그 노드의 Flow만 가짐 (`:4244`)
- Hubble Relay: 모든 노드의 Flow를 모아 클러스터 전체를 보여 줌 (`:4245`)
- hubble CLI / UI: Relay에 붙어서 조회

### eBPF Event에서 Hubble Flow까지

<img width="1360" height="1304" alt="image" src="https://github.com/user-attachments/assets/983131e9-db93-4627-90eb-0ef50f44cad6" />


- BPF 프로그램이 trace / drop / policy-verdict 이벤트를 perf ring buffer(`cilium_events`)에 남김
- cilium-agent의 monitor가 읽어서 Hubble에 넘김
- Hubble이 Flow로 변환하면서 Pod 이름, namespace, label을 붙임
    - 이벤트 자체에는 IP와 identity 숫자 정도만 있음
- Flow는 노드별 ring buffer에 저장 (기본 4095개, 차면 오래된 것부터 덮어씀)
- 이 구조 때문에 생기는 특징
    - Hubble은 패킷을 따로 캡처하지 않음 → datapath가 남긴 이벤트를 보여 주는 것
    - Cilium이 관리하지 않는 Pod(hostNetwork 등)는 Hubble에도 안 보임
    - 기본 설정에선 연결마다 일부 패킷만 trace로 남음 (새 연결, 새 TCP 플래그, 약 5초마다) → drop은 항상 남음
    - `cilium-dbg monitor` 도 같은 이벤트를 봄 → Hubble이 안 될 때 대신 사용 가능

### FORWARDED / DROPPED

- FORWARDED
    - 정상이면 TCP handshake가 FORWARDED로 이어짐
    - 다른 노드로 가면 출발 노드는 `to-overlay`, 도착 노드는 `to-endpoint` 로 보임
- DROPPED: drop reason부터 확인
    - `Policy denied`: NetworkPolicy에 걸림 (가장 흔함)
    - `CT: Map insertion failed`: CT Map이 가득 참
    - `Service backend not found`: Service에 backend가 없음
    - `Stale or unroutable IP`: 사라진 Pod IP로 보냄
- L4 정책에 막히면 SYN이 재전송마다 DROPPED로 반복됨, L7 정책에 막히면 `http-request DROPPED` (클라이언트는 403)

### Hubble로 통신 문제 추적하기

<img width="1360" height="720" alt="image" src="https://github.com/user-attachments/assets/58dbfcee-a61a-4a28-b940-65410a89485f" />


1. 범위 좁히기: `hubble observe --from-pod <ns>/<pod> --to-pod <ns>/<pod>`
2. 막힌 것만 보기: `-verdict DROPPED` → drop reason 확인
3. `Policy denied` 면 어느 쪽에서 막혔는지 구분
    - `traffic_direction` 이 EGRESS → 출발지 Pod의 egress 정책
    - INGRESS → 목적지 Pod의 ingress 정책
4. 정책이 실제로 어떻게 들어갔는지 확인: `cilium-dbg endpoint get <id>`, `cilium-dbg bpf policy get <id>`
5. DROPPED가 없는데 안 될 때
    - Flow 자체가 없음 → DNS 단계 실패, Cilium이 못 보는 경로
    - 출발 노드 `to-overlay` 만 있고 도착 노드엔 없음 → 노드 간 네트워크·방화벽
    - 응답까지 FORWARDED → 애플리케이션 문제
- 자주 하는 실수: egress 정책을 걸면서 DNS(kube-dns:53) 허용을 빠뜨림 → 이름 해석부터 DROPPED
