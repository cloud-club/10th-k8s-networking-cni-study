# Week 8. Kubernetes & CNI Troubleshooting

Calico와 Cilium의 대표적인 네트워크 장애를 분석하고, 원인에 맞는 복구 방법과 변경 작업의 영향 범위를 학습합니다.

## 1. Calico — 노드 간 Pod 통신 장애

### 1.1 장애 증상

- 같은 노드에 배치된 Pod끼리는 통신 가능함
- 서로 다른 노드에 배치된 Pod끼리는 통신 실패함
- Pod는 `Running` 상태이며, IP도 정상 할당되어 있음

### 1.2 원리

Calico는 설정된 데이터패스 방식에 따라 Pod 패킷을 전달함.

```text
Pod A (Node 1)
      |
      v
Calico Datapath
      |
      +-- BGP / Native Routing
      |       Node 2의 Pod CIDR로 직접 라우팅
      |
      +-- VXLAN / IP-in-IP
              Pod 패킷을 캡슐화
              Node 2의 Underlay IP로 전달
                      |
                      v
               Pod B (Node 2)
```

같은 노드 안에서는 노드 간 네트워크를 거치지 않고 통신할 수 있지만, 다른 노드의 Pod로 가려면 노드 간 라우팅이나 캡슐화 경로가 정상이어야 함.

### 1.3 발생 원인

- **BGP 문제**: BGP Peer가 끊기거나 원격 Pod CIDR 경로를 학습하지 못함
- **VXLAN 문제**: 노드 간 UDP 4789 통신이 방화벽에서 차단됨
- **IP-in-IP 문제**: IP 프로토콜 4번 트래픽이 차단됨
- **Underlay 문제**: 노드 IP 간 라우팅이 잘못되었거나 네트워크 ACL이 트래픽을 차단함
- **MTU 문제**: 캡슐화 헤더 때문에 패킷 크기가 경로의 MTU를 초과함

### 1.4 진단 및 해결

```bash
# Pod가 어느 노드에 배치되었는지 확인
kubectl get pods -o wide -A

# 노드의 라우팅 테이블 확인
ip route

# Calico 노드 로그 확인
kubectl logs -n calico-system <calico-node-pod>

# BGP 기반 구성에서만 확인
calicoctl node status

# VXLAN 패킷 확인
tcpdump -ni any udp port 4789
```

| 관찰 결과 | 조치 |
|---|---|
| 원격 Pod CIDR 경로가 없음 | BGP Peer 및 Route 광고 상태 확인 |
| VXLAN 패킷이 송신 노드에서 나가지 않음 | Calico 설정, 라우트, 터널 인터페이스 확인 |
| 패킷이 송신 노드에서 나가지만 수신 노드에 도착하지 않음 | Underlay 라우팅, 방화벽, 보안 그룹 및 ACL 확인 |
| 큰 패킷만 실패함 | 경로 MTU, 캡슐화 오버헤드, PMTUD 동작 확인 |

**해결 원칙:** 실제로 사용하는 라우팅 방식에 해당하는 문제부터 수정함. BGP를 사용하지 않는 VXLAN 구성에서 BGP Peer를 점검하거나, 근거 없이 캡슐화 방식을 변경하지 않음.

### 1.5 해결 시 Side Effect 및 영향 범위

| 조치 | 영향 및 이유 |
|---|---|
| 방화벽 규칙 수정 | 허용 범위가 넓어지면 다른 노드나 워크로드에도 접근 범위가 확대될 수 있음 |
| BGP 설정 수정 | Route 광고 및 철회가 발생하면 해당 경로를 사용하는 다른 노드의 통신에도 영향이 생길 수 있음 |
| VXLAN/IP-in-IP 설정 변경 | 노드 간 데이터패스가 바뀌면서 기존 경로와 MTU 가정이 달라질 수 있음 |
| MTU 조정 | 변경 대상 인터페이스와 Pod 네트워크의 MTU 불일치가 생기면 일부 통신이 계속 실패할 수 있음 |

단일 노드의 잘못된 라우트만 수정하면 영향 범위가 제한될 수 있음. 반면 공통 IPPool, 캡슐화 설정, BGP 광고 정책을 변경하면 여러 노드에 영향을 줄 수 있음.

### 1.6 사전 예방

- Calico IPPool의 `encapsulation` 및 `natOutgoing` 설정을 변경 이력과 함께 관리함
- BGP 사용 시 Peer 상태와 Route 광고를 모니터링함
- VXLAN/IP-in-IP에 필요한 프로토콜과 포트가 노드 간에 허용되는지 검증함
- Pod MTU와 Underlay MTU를 고려해 캡슐화 오버헤드를 반영함
- 배포 전후에 동일 노드 및 노드 간 Pod 통신 테스트를 수행함

---

## 2. Cilium — eBPF Datapath Drop 장애

### 2.1 장애 증상

- Pod가 정상 실행 중인데 특정 통신만 실패함
- 일부 목적지 또는 포트에서만 연결이 차단됨
- Hubble에서 `DROPPED` Flow가 관찰됨
- 정책을 수정했는데도 통신이 복구되지 않거나, 간헐적으로 연결이 실패함

### 2.2 원리

Cilium은 eBPF 프로그램을 이용해 패킷 처리 경로에서 정책 적용, Service 처리 및 연결 추적 등을 수행할 수 있음.

```text
Pod에서 패킷 발생
       |
       v
Cilium eBPF Datapath
       |
       +-- NetworkPolicy 검사
       |
       +-- Service / Routing 처리
       |
       +-- Connection Tracking
       |
       v
Forward 또는 Drop
       |
       v
Hubble에서 관측 가능한 Flow / Drop Event
```

Hubble은 네트워크 흐름을 관찰하는 도구이며, Drop 이벤트를 통해 차단 원인을 좁힐 수 있음. 다만 모든 통신 실패가 eBPF 정책에 의한 Drop인 것은 아님.

### 2.3 발생 원인

- **NetworkPolicy**: 목적지, 포트, 방향 또는 보안 Identity에 대한 허용 규칙이 없음
- **Identity / Endpoint 상태**: 정책에 사용된 Pod 선택 조건이나 Identity가 기대와 다름
- **Routing / Service 문제**: 목적지 경로 또는 Service Backend 연결이 잘못됨
- **Connection Tracking 문제**: CT Map 삽입 실패나 테이블 용량 부족 등으로 연결 상태를 정상적으로 처리하지 못함

### 2.4 진단 및 해결

```bash
# Cilium 상태 확인
cilium status

# Hubble에서 Drop된 Flow 확인
hubble observe --verdict DROPPED

# eBPF Drop 이벤트 확인
cilium-dbg monitor --type drop

# 적용된 정책 확인
kubectl get networkpolicy,ciliumnetworkpolicy -A

# Cilium Service 상태 확인
cilium service list
```

| 관찰 결과 | 조치 |
|---|---|
| `Policy denied` 등 정책 관련 Drop 사유 확인 | Ingress/Egress 정책, Pod selector, Identity 및 포트 조건 확인 |
| Service 경유 통신만 실패함 | Service 및 Backend 상태, Cilium Service 정보 확인 |
| `CT: Map insertion failed` 등 CT 관련 오류 확인 | CT Map 사용량과 관련 메트릭·로그를 확인하고 용량 또는 연결 패턴 문제를 조사 |
| Drop 이벤트가 없지만 통신이 실패함 | DNS, 애플리케이션 Listen 상태, Routing, 반환 경로 및 관측 지점 확인 |

**해결 원칙:** Drop 사유를 먼저 확인한 후 해당 원인을 수정함. 통신이 실패한다는 이유만으로 모든 NetworkPolicy를 제거하거나 Cilium을 재설치하지 않음.

### 2.5 해결 시 Side Effect 및 영향 범위

| 조치 | 영향 및 이유 |
|---|---|
| NetworkPolicy 수정 | 허용 범위가 넓어지면 다른 Pod에도 의도하지 않은 통신이 허용될 수 있음 |
| Cilium 공통 설정 변경 | 여러 Endpoint의 데이터패스 동작이나 정책 처리에 영향을 줄 수 있음 |
| CT Map 용량 조정 | 메모리 사용량이 증가할 수 있으며, 재시작이나 연결 상태 초기화가 동반되는 작업은 기존 연결에 영향을 줄 수 있음 |
| Cilium Agent 재시작 | 노드 단위의 데이터패스 재구성이나 일시적인 관측 공백이 발생할 수 있음. 영향은 설정과 환경에 따라 달라짐 |

정책 하나를 정확하게 수정하면 영향 범위를 좁힐 수 있음. 반대로 공통 정책이나 Agent 설정을 변경하면 해당 노드의 여러 Endpoint 또는 클러스터 내 다수 워크로드에 영향이 확대될 수 있음.

### 2.6 사전 예방

- NetworkPolicy를 최소 권한 원칙에 따라 설계하고 변경 후 허용·차단 테스트를 수행함
- Hubble Drop 이벤트와 Cilium Agent 로그를 모니터링함
- CT Map 관련 메트릭과 오류 로그를 점검해 용량 부족 징후를 확인함
- Cilium 설정 변경 시 적용 대상과 롤백 방법을 사전에 정의함
- 변경 전후에 정책 적용 상태와 실제 Pod 간 통신을 검증함

---

## 3. 공통 Troubleshooting 원칙

두 CNI 모두 다음 순서로 장애를 분석함.

1. **증상 분리**: 동일 노드와 노드 간 통신, 특정 포트와 전체 통신을 구분함
2. **구성 확인**: Pod IP, 노드 배치, 라우팅 모드, 정책 및 Service 상태를 확인함
3. **데이터패스 추적**: Route, 터널 패킷, eBPF Drop Event 등 실제 증거를 수집함
4. **최소 범위 수정**: 확인된 원인만 수정하고 공통 설정 변경은 영향 범위를 검토함
5. **복구 검증**: 정상 통신뿐 아니라 기존 연결, 다른 노드, 정책 동작도 확인함
6. **재발 방지**: 모니터링, 통신 테스트, 설정 검증 및 롤백 절차를 마련함

## 핵심 비교

| 구분 | Calico Routing 장애 | Cilium Datapath Drop 장애 |
|---|---|---|
| 주요 조사 대상 | BGP, Route, VXLAN/IP-in-IP, Underlay | eBPF Drop Event, NetworkPolicy, Identity, CT Map |
| 대표적인 관측 도구 | `ip route`, `calicoctl`, `tcpdump` | Hubble, `cilium-dbg`, Cilium 메트릭 |
| 주요 위험 | Route 변경 및 캡슐화 설정 변경의 영향 | 정책 범위 확대, 공통 설정 변경, 연결 상태 영향 |
| 예방 핵심 | 라우팅 및 MTU 검증 | 정책 검증과 Datapath 관측 |

## 참고 자료

- Calico Troubleshooting: https://docs.tigera.io/calico/latest/operations/troubleshoot/troubleshooting
- Calico Troubleshooting Commands: https://docs.tigera.io/calico/latest/operations/troubleshoot/commands
- Cilium Troubleshooting: https://docs.cilium.io/en/stable/operations/troubleshooting/
- Cilium Network Policy: https://docs.cilium.io/en/stable/security/policy/
- Cilium BPF Maps: https://docs.cilium.io/en/stable/network/ebpf/maps/