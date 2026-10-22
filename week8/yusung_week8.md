## 1. 장애를 어디서부터 분해해서 볼지

<img width="2720" height="1120" alt="image" src="https://github.com/user-attachments/assets/f5e8d26b-4e9b-4f9b-aacb-9f1f4b554ee5" />


- 요청 경로를 구간으로 쪼개서 **어디까지 되는지** 확인
- 구간: DNS → Service → Routing → Policy → App
- 통신 종류(Pod-to-Pod / Pod-to-Service / External)에 따라 지나가는 구간이 달라짐

### 먼저 확인할 3가지

- 범위: 어디서 안 되는지
    - 특정 Pod만 → 그 Pod의 설정·정책·앱
    - 특정 노드만 → 그 노드의 CNI agent, kube-proxy, 라우팅
    - 클러스터 전체 → CoreDNS, CNI control plane, 최근 변경
- 증상: timeout인지, refused인지, 이름 해석 실패인지
- 변경: 직전에 배포·정책·업그레이드·노드 추가가 있었는지

### 에러 메시지로 구간 추정

| 증상 | 의미 | 먼저 볼 곳 |
| --- | --- | --- |
| `could not resolve host` / `NXDOMAIN` | 이름 해석 실패 | CoreDNS, `/etc/resolv.conf`, DNS 허용 정책 |
| `connection refused` | 목적지까지 갔는데 받는 쪽이 없음 | targetPort, 앱 listen 주소, Endpoint 비어 있음 |
| `timeout` | 패킷이 중간에서 사라짐 | NetworkPolicy, 노드 간 라우팅, 방화벽·SG, MTU |
| 403 / 503 | L7까지는 도달함 | L7 정책, Ingress, 앱 |
| 가끔만 실패 | 일부 backend·노드만 문제 | 특정 Endpoint, conntrack, DNS 5초 지연 |

### 기본 점검 순서

<img width="2720" height="1520" alt="image" src="https://github.com/user-attachments/assets/cff3cb90-c5e6-49d8-9832-9afa98182248" />


1. Pod 상태: `kubectl get pod -o wide` → Running/Ready, IP, 노드
2. DNS: `nslookup <svc>.<ns>.svc.cluster.local`
3. Endpoint: `kubectl get endpointslice -l kubernetes.io/service-name=<svc>`
4. Service 우회: `curl <podIP>:<port>` → 되면 Service 쪽, 안 되면 Pod·정책 쪽
5. 같은 노드 Pod ↔ 다른 노드 Pod 비교 → 다른 노드만 안 되면 노드 간 구간
6. 정책: `kubectl get netpol -A`, `hubble observe --verdict DROPPED`
7. 노드 레벨: `ip route`, CNI agent 로그, kube-proxy 로그
- 핵심: Service 이름 → ClusterIP → Pod IP 순으로 **한 겹씩 벗겨 가며** 어디서 끊기는지 찾기
- 디버그 도구
    - 일회성 Pod: `kubectl run tmp --rm -it --image=nicolaka/netshoot -- bash`
    - 문제 Pod 안에서: `kubectl debug -it <pod> --image=nicolaka/netshoot` (같은 network namespace 사용)
    - 앱 이미지에 curl·nslookup이 없는 경우가 많아서 필요함

---

## 2. 통신 종류별 장애 분석

### Pod-to-Pod

<img width="2720" height="1120" alt="image" src="https://github.com/user-attachments/assets/37b2ceab-a5f8-4530-8523-069f962da79b" />


- 경로: Pod eth0 → veth(호스트 쪽) → 노드 라우팅 → (다른 노드면) 터널·라우트 → 상대 노드 → veth → Pod
- 같은 노드는 되는데 **다른 노드만** 안 될 때 → 노드 간 구간 문제
    - overlay: 터널 포트가 방화벽·SG에 막힘 (VXLAN UDP 8472, Geneve UDP 6081, Calico IPIP는 IP 프로토콜 4)
    - native routing: 노드에 상대 Pod CIDR 라우트가 없음, 클라우드 source/dest check
    - MTU: 캡슐화 헤더(VXLAN 50바이트)만큼 작아짐 → 작은 요청은 되고 큰 응답에서 멈춤
- 어디서든 안 될 때 → 목적지 Pod 자체 또는 default deny 정책
- 확인
    - `ip route get <podIP>`: 어느 인터페이스, 어느 next hop으로 나가는지
    - `cilium-health status`: 노드 간 연결 상태
    - `tcpdump -ni <iface> udp port 8472`: 터널 패킷이 실제로 나가고 들어오는지

### Pod-to-Service

<img width="2720" height="1040" alt="image" src="https://github.com/user-attachments/assets/c3894897-5da9-4f0e-bfa7-9ca8b4099ef7" />


- Pod가 Service 이름으로 요청할 때의 기본 흐름
    1. DNS 조회: service → ClusterIP (예: 10.96.10.20)
    2. Pod가 ClusterIP로 패킷을 보냄
    3. 출발 노드에서 목적지를 ClusterIP → 실제 Pod IP로 바꿈 (DNAT)
        - 바꾸는 주체: kube-proxy가 만들어 둔 iptables/IPVS 규칙, 또는 Cilium eBPF
    4. 바뀐 Pod IP로 전달 → 여기부터는 Pod-to-Pod와 같음
- ClusterIP는 어느 인터페이스에도 없는 가상 IP → ping이 안 되는 게 정상 (포트로 테스트)
- 단계별 확인
    1. DNS: 이름이 ClusterIP로 풀리는지
    2. Endpoint가 비어 있는지
        - Service `selector` 와 Pod label 불일치
        - readinessProbe 실패 → NotReady Pod는 Endpoint에서 빠짐
    3. 포트 연결: Service `port` → `targetPort` → 컨테이너가 실제 listen 하는 포트 (named port 오타 주의)
    4. Pod IP로 직접 접근: 되면 Service 구현(kube-proxy/eBPF LB) 문제
    5. Service 규칙: `iptables-save -t nat | grep <svc>`, `ipvsadm -Ln`, `cilium-dbg service list`
- 앱이 `127.0.0.1` 에만 listen → Pod 안에서 curl하면 되는데 밖에서는 refused

### External

<img width="2720" height="1280" alt="image" src="https://github.com/user-attachments/assets/a74b9a39-c547-4274-904c-a4e4a619971f" />


- 나가는 방향 (Pod → 외부)
    - Pod IP가 노드 IP로 SNAT(masquerade) 돼서 나감
    - 안 되면: egress NetworkPolicy, masquerade 설정, 노드 자체의 외부 경로(NAT GW), DNS upstream
    - 외부 쪽 allowlist에는 Pod IP가 아니라 노드 IP(또는 NAT GW IP)가 보임 → 노드가 늘면 allowlist도 깨짐 → Egress Gateway로 출발 IP 고정
- 들어오는 방향 (외부 → Pod)
    - LoadBalancer → NodePort → Pod, 또는 Ingress Controller → Service → Pod
    - `EXTERNAL-IP` 가 `<pending>` → LB를 만들어 줄 구현(클라우드 컨트롤러, MetalLB 등)이 없음
    - NodePort 범위(30000–32767)가 노드 방화벽에 막힘
    - Pod에 ingress 정책이 있는데 Ingress Controller namespace를 허용하지 않음
- `externalTrafficPolicy`

| 값 | 동작 | 주의 |
| --- | --- | --- |
| Cluster (기본) | 아무 노드로 들어와도 다른 노드 Pod로 넘겨 줌 | 넘길 때 SNAT → 앱에서 클라이언트 IP가 노드 IP로 보임 |
| Local | Pod가 있는 노드에서만 처리 | 클라이언트 IP 보존, Pod 없는 노드로 오면 drop → LB health check로 빼야 함 |

---

## 3. CNI 장애 Troubleshooting

### CNI 동작

<img width="2720" height="1200" alt="image" src="https://github.com/user-attachments/assets/8cd2f2ca-d3bd-4efa-b1cc-765f380b6904" />


- Pod 생성 시: kubelet → containerd(CRI)가 netns 생성 → CNI 플러그인 실행 (CNI ADD)
    - 설정은 `/etc/cni/net.d/`, 바이너리는 `/opt/cni/bin/`
    - 플러그인이 veth 생성, IPAM으로 IP 받기, 라우트 설정 → 결과를 런타임에 돌려줌
- Pod 삭제 시: CNI DEL → IP 반환, veth 정리
- CNI는 두 부분으로 나뉨
    - 바이너리: Pod 생성·삭제 시점에만 실행됨
    - agent(DaemonSet): 설정 파일 배치, datapath·정책 프로그래밍 (Cilium agent, calico-node 등)
    - 둘 중 하나만 문제여도 실패

### 증상별 원인

| 증상 | 원인 | 확인 |
| --- | --- | --- |
| Node `NotReady` + `cni config uninitialized` | `/etc/cni/net.d/` 에 설정 없음 (agent가 아직 안 뜸) | CNI DaemonSet Pod 상태 |
| Pod `ContainerCreating` + `FailedCreatePodSandBox` | CNI ADD 실패 (IP 없음, agent 응답 없음 등) | `kubectl describe pod` Events |
| 기존 Pod는 되는데 새 Pod만 통신 안 됨 | agent 다운 → 새 endpoint가 datapath에 반영 안 됨 | agent 로그, `cilium-dbg endpoint list` |
| 의도와 다른 CNI가 동작 | `/etc/cni/net.d/` 에 설정이 여러 개 → 이름순 첫 파일 사용 | 파일 목록 |
| CNI 교체 후 이상 동작 | 이전 CNI 잔재 (iptables 규칙, 인터페이스) | `ip link`, `iptables-save` |
- 로그 위치
    - kubelet: `journalctl -u kubelet | grep -i cni`
    - Cilium: `cilium status`, `cilium-dbg status --verbose`, `kubectl -n kube-system logs ds/cilium`
- 전체 점검: `cilium connectivity test` (Pod-Pod, Service, 외부, 정책까지 한 번에 테스트)

---

## 4. IPAM / IP Pool 문제 분석

### IPAM 방식

<img width="2720" height="1120" alt="image" src="https://github.com/user-attachments/assets/16d03548-96ba-4695-9f18-45f8e6c78a9f" />


- IPAM: Pod에 줄 IP를 어디서 어떻게 나눠 주는지 관리

| 방식 | 할당 구조 | 예 |
| --- | --- | --- |
| Kubernetes host-scope | 노드마다 `spec.podCIDR` 을 받아 그 안에서 할당 | Flannel, Cilium `kubernetes` 모드 |
| cluster-pool | CNI가 클러스터 풀에서 노드별 CIDR를 나눠 줌 | Cilium 기본 (10.0.0.0/8, 노드당 /24) |
| IP Pool (block) | 풀에서 블록 단위로 노드에 붙임 | Calico IPPool (기본 /26 블록) |
| VPC 네이티브 | 클라우드 VPC IP를 ENI로 받아 Pod에 직접 줌 | AWS VPC CNI, Cilium ENI 모드 |

### 자주 발생하는 문제

- 노드 IP 고갈
    - 노드당 /24면 약 254개 → 그 노드에서 더 이상 Pod 생성 불가
    - 증상: `ContainerCreating` + `no IP addresses available` 류 에러
- IP leak
    - 노드 비정상 종료 등으로 CNI DEL이 실패 → 할당 기록만 남고 반환 안 됨
    - 실제 Pod 수보다 사용 중 IP가 많아짐 → 결국 고갈
- 클라우드 VPC 모드
    - 서브넷 IP 부족, 인스턴스 타입별 ENI·IP 수 한도 → 노드당 Pod 수 제한
- CIDR 겹침
    - Pod CIDR / Service CIDR / 노드 대역 / 사내망·VPN 대역이 겹침 → 특정 대상만 이상하게 안 됨
    - 설치 후 바꾸기 어려움 → 설계 단계에서 대역 확인 필수
- 확인
    - `kubectl get node -o custom-columns=NAME:.metadata.name,CIDR:.spec.podCIDR`
    - Cilium: `kubectl get ciliumnode <node> -o yaml` (`spec.ipam.podCIDRs`), `cilium-dbg status --all-addresses`
    - Calico: `calicoctl ipam show --show-blocks`, `calicoctl ipam check` (leak 확인)

---

## 5. Routing / NetworkPolicy / DNS / kube-proxy 문제 분석

### Routing

- 노드 간 Pod 트래픽 전달 방식
    - overlay: 터널로 감싸서 노드 IP로 전달 → 터널 포트, MTU 문제
    - native routing: Pod CIDR 라우트를 노드·라우터에 직접 설정 (BGP, 클라우드 라우트 테이블) → 라우트 누락 문제
- 흔한 원인
    - 노드 추가 후 라우트 전파 안 됨 → BGP 피어 상태 (`cilium bgp peers`, `calicoctl node status`)
    - 비대칭 경로 → stateful 방화벽·conntrack에서 응답이 drop
    - `rp_filter` 엄격 모드 → 들어온 인터페이스와 응답 경로가 다르면 drop
    - MTU 불일치 → 큰 패킷만 사라짐

### NetworkPolicy

- 규칙 정리
    - 정책이 하나라도 Pod를 선택하면, 그 방향(ingress/egress)은 default deny
    - 표준 NetworkPolicy는 **허용만** 있음 → 여러 정책은 허용 규칙의 합집합, 순서 없음
    - ingress와 egress는 따로 → A egress 허용 + B ingress 허용이 둘 다 있어야 통신됨
    - CNI가 정책을 구현해야 동작 (Flannel 단독은 미지원 → 정책을 만들어도 효과 없음)
- 자주 하는 실수
    - egress 정책에서 DNS(kube-dns, UDP/TCP 53) 허용 누락 → 이름 해석부터 실패
    - `namespaceSelector` 와 `podSelector` 의 AND / OR 혼동 ( 하나 차이)
    - namespace에 label이 없음 → `kubernetes.io/metadata.name` 자동 label 사용
    - Pod를 `ipBlock` 으로 지정 → Pod IP는 바뀜, Pod는 label로 지정
- 확인: `kubectl describe netpol <name>` → Hubble `-verdict DROPPED` 에서 `Policy denied` + 방향 확인 (Week 7 참고)

### kube-proxy

- 역할: Service → Endpoint 정보를 노드의 커널 규칙(iptables / IPVS / nftables)으로 바꿔 줌
- kube-proxy가 직접 패킷을 넘기는 게 아니라 **규칙만 써 둠** → kube-proxy가 죽어도 기존 규칙으로 트래픽은 흐름 (새 변경만 반영 안 됨 → 늦게 발견됨)

| 모드 | 방식 | 특징 |
| --- | --- | --- |
| iptables | NAT 체인, 확률로 backend 선택 | 기본값, Service가 많으면 규칙 갱신이 느림 |
| IPVS | 커널 L4 LB, 해시 조회 | `ipvsadm -Ln` 으로 확인, 1.35부터 deprecated |
| nftables | nftables 기반 | 1.33 GA, iptables의 후속 |
| 없음 (eBPF) | Cilium kube-proxy replacement | `cilium-dbg service list` 로 확인 |
- 자주 터지는 케이스
    - kube-proxy replacement를 켰는데 kube-proxy도 남아 있음 → 규칙이 이중으로 생김
    - conntrack 테이블 가득 → `dmesg` 에 `nf_conntrack: table full, dropping packet` → `nf_conntrack_max` 조정 (Cilium은 CT Map 크기)
    - stale conntrack: UDP Service의 backend가 바뀌었는데 옛 Pod IP로 계속 감 → `conntrack -D` 로 항목 삭제
- 확인
    - `kubectl -n kube-system logs ds/kube-proxy`
    - 노드에서 `curl localhost:10249/proxyMode` → 현재 모드
    - `iptables-save -t nat | grep KUBE-SVC`

---

## 6. 운영 중 자주 터지는 케이스

| 케이스 | 증상 | 원인 | 대응 |
| --- | --- | --- | --- |
| 정책 적용 직후 전체 장애 | 이름 해석 실패 | egress에 DNS 허용 누락 | kube-dns UDP/TCP 53 허용 |
| Service 호출 실패 | refused / 503 | Endpoint 없음 (label 불일치, readiness 실패) | selector·probe 확인 |
| 배포 때마다 일부 요청 실패 | 롤링 업데이트 중 502 | 종료 중 Pod로 트래픽이 아직 감 | `preStop` sleep, graceful shutdown |
| 다른 노드 Pod만 불통 | timeout | 터널 포트·SG 차단, 라우트 누락 | 8472 등 허용, BGP 상태 확인 |
| 큰 응답만 멈춤 | 작은 요청은 정상 | MTU | CNI MTU를 실제 경로에 맞춤 |
| 새 Pod가 안 뜸 | `ContainerCreating` | IP 고갈, CNI agent 장애 | IP pool 확장, agent 복구 |
| DNS 간헐 지연 | p99 latency 5초 | conntrack race, CoreDNS 과부하 | NodeLocal DNSCache, CoreDNS 확장 |
| 트래픽 많은 노드에서 랜덤 실패 | 간헐 timeout | conntrack table full | `nf_conntrack_max` / CT Map 크기 |
| 외부 API 호출 차단 | 외부에서 403 / timeout | 출발 IP가 노드 IP, allowlist 불일치 | Egress Gateway, NAT GW 고정 IP |
| 클라이언트 IP가 안 보임 | 로그에 노드 IP | `externalTrafficPolicy: Cluster` 의 SNAT | `Local` + LB health check |

### 대응 흐름 정리

1. 재현할 Pod 하나를 고정 (출발지·목적지 명확히)
2. 범위 확인: Pod / 노드 / 클러스터 전체 중 어디인지
3. 에러 메시지로 구간 추정 (resolve / refused / timeout / 4xx·5xx)
4. 한 겹씩 우회: 이름 대신 IP, Service 대신 Pod IP, 다른 노드 대신 같은 노드
5. 끊기는 구간에서 Hubble DROPPED, 노드 라우트, agent 로그 확인
- 평소에 준비해 둘 것
    - netshoot 같은 디버그 이미지를 클러스터에서 바로 쓸 수 있게
    - 알람: CoreDNS 에러율·지연, IP pool 사용률, conntrack 사용률, CNI agent 상태
    - CIDR 대역표 (Pod / Service / 노드 / 사내망) 문서화
