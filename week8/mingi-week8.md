# Kubernetes Networking Common Issue Troubleshooting & Operations - week 8

## 장애를 어디서부터 분해해서 볼까?

### 통신 경로 구분

| 통신 유형 | 주요 확인 대상 |
| --- | --- |
| **Pod → Pod: 같은 Node** | 애플리케이션 리스닝, veth·로컬 전달 경로, 정책 |
| **Pod → Pod: 다른 Node** | 로컬 경로 + 노드 간 라우팅·터널·방화벽·반환 경로 |
| **Pod → Service** | backend Pod + selector·EndpointSlice·포트·Service 구현 |
| **Pod → DNS** | resolver 설정, DNS 서버까지의 통신, CoreDNS·상위 DNS |
| **Pod → External** | 이름 해석, egress 정책, 라우팅, 필요한 SNAT·NAT Gateway, 반환 경로 |
| **External → Service / Ingress** | LB·NodePort, Ingress Controller, backend, 정책·트래픽 정책 |

**Pod → DNS도 하나의 네트워크 통신이다. 이름 해석 실패가 항상 CoreDNS 설정 오류를 뜻하지는 않는다.**

### 먼저 기록할 정보

- **출발지:** Pod·Namespace·Node·출발지 IP
- **목적지:** 이름·실제 IP·포트·TCP/UDP
- **오류:** DNS 오류, timeout, connection refused, connection reset, HTTP 상태
- **범위:** 특정 Pod / 특정 Node / 특정 Namespace / 전체
- **조건:** 항상 / 간헐적 / 큰 응답 / 동시 연결 증가 / 재배포 직후
- **변경:** 애플리케이션, NetworkPolicy, CNI, OS·커널, 방화벽, 클러스터 업그레이드

| 관찰 | 다음에 확인할 대상 |
| --- | --- |
| 대상 Pod IP도 실패 | 앱 리스닝, 정책, Pod 경로 |
| 모든 backend Pod IP는 성공, Service IP는 실패 | Service 설정·트래픽 정책·Service 구현 |
| Service IP는 성공, 이름으로는 실패 | DNS 질의 결과와 애플리케이션의 이름 사용 |
| 같은 Node에서는 성공, 다른 Node에서는 실패 | 노드 간 경로·터널·기반망 |
| 기존 Pod는 정상, 새 Pod만 네트워크 생성 실패 | CNI 호출·IPAM·노드별 설정 |
| 작은 요청은 정상, 큰 전송만 실패 | MTU·PMTUD·손실·부하 조건 |

이 표는 **조사 우선순위**이다. 하나의 결과만으로 원인이 확정되는 것은 아니다.

---

## 기본 점검 순서

**상태 확인 → Pod 직접 접속 → Service 접속 → DNS 확인 → 실패 구간의 경로·정책 조사 → 원래 조건으로 복구 검증**

### 1. Pod·Node·이벤트 확인

관리용 터미널에서 실행한다.

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide

kubectl -n <namespace> describe pod <problem-pod>
kubectl -n <namespace> get events --sort-by=.metadata.creationTimestamp

kubectl -n <namespace> logs <problem-pod> -c <container> \
  --since=15m --timestamps
```

- `Pending`이면 Node 배정 여부와 스케줄링 이벤트부터 확인
- `ContainerCreating`이면 이미지·볼륨·sandbox 등 실제 이벤트 확인
- `FailedCreatePodSandBox`이면 메시지 안의 CNI·IPAM 오류 확인
- `Running`이어도 readiness와 실제 애플리케이션 응답 확인
- 특정 Node에 실패가 모이는지 Pod 배치와 함께 비교

**sandbox 생성 실패라는 상위 메시지만 보고 IP 부족으로 단정하지 않는다. 하위 오류와 발생 Node를 함께 확인한다.**

### 2. 같은 출발지에서 접속 방식 비교

예시 환경은 다음과 같다. 실제 값은 `kubectl get pods -o wide`, `kubectl get svc`로 확인한다.

| 대상 | 설명용 값 |
| --- | --- |
| Namespace / 출발 Pod | `netlab` / `client` |
| 같은 Node의 backend | `10.244.1.20:8080` |
| 다른 Node의 backend | `10.244.2.20:8080` |
| Service | `api` / `10.96.0.100:80` |
| Service 연결 | `port: 80 → targetPort: 8080` |
| 클러스터 DNS 도메인 | `cluster.local` 가정 |

```bash
# 출발 Pod 안에서 같은 Node / 다른 Node의 backend로 직접 요청
kubectl -n netlab exec client -- \
  curl -sv --connect-timeout 3 --max-time 5 <http://10.244.1.20:8080/>

kubectl -n netlab exec client -- \
  curl -sv --connect-timeout 3 --max-time 5 <http://10.244.2.20:8080/>

# 같은 출발 Pod에서 Service IP로 요청
kubectl -n netlab exec client -- \
  curl -sv --connect-timeout 3 --max-time 5 <http://10.96.0.100:80/>

# DNS 질의와 이름을 통한 실제 접속을 각각 확인
kubectl -n netlab exec client -- nslookup api.netlab.svc.cluster.local

kubectl -n netlab exec client -- \
  curl -sv --connect-timeout 3 --max-time 5 <http://api.netlab.svc.cluster.local/>
```

- backend는 하나만 확인하지 않고 실제 대상들을 각각 비교
- TCP/UDP와 목적지 포트를 맞춰 확인
- HTTP Host·TLS SNI가 필요한 서비스는 IP로 바꿀 때도 해당 조건 유지
- 별도의 진단 Pod는 Node·label·정책이 달라질 수 있으므로 원래 출발 Pod의 조건과 비교
- `ping`은 ICMP 확인이다. 성공해도 실제 TCP 포트의 정상 동작을 보장하지 않음

---

## Pod-to-Pod / CNI / Routing 장애

### CNI 구성 실패와 전달 경로 장애

**Pod 네트워크를 만드는 과정과, 만들어진 네트워크에서 패킷을 전달하는 과정을 나누어 확인한다.**

```
Pod 생성 시
kubelet → Container Runtime → CNI Plugin → IPAM·인터페이스·경로 설정

통신 시
Pod → 이미 구성된 커널·eBPF·NIC 경로 → 목적지
```

| 구분 | 대표 증상 | 확인할 것 |
| --- | --- | --- |
| **CNI 호출·구성 실패** | 새 Pod의 sandbox 생성 실패 | CNI 설정·바이너리·권한, Agent/API 접근, IPAM 오류 |
| **데이터 패스 장애** | Pod는 실행되지만 통신 실패 | 라우팅, 터널, 정책, 방화벽, MTU, 반환 경로 |
| **특정 Node의 장애** | 그 Node의 Pod에 실패 집중 | 해당 Node의 Agent·NIC·설정·경로와 정상 Node 비교 |

```bash
# 관리용 터미널: 설치된 CNI와 문제 Node의 Agent 확인
kubectl get daemonsets -A
kubectl get pods -A -o wide --field-selector spec.nodeName=<node>

kubectl -n <cni-namespace> logs <cni-pod> -c <container> \
  --since=15m --timestamps

# 문제 Node 호스트: 일반적인 CNI 경로와 kubelet 로그
ls -l /etc/cni/net.d /opt/cni/bin
sudo journalctl -u kubelet --since "15 min ago"
```

CNI 경로·서비스 이름은 런타임과 배포판에 따라 달라진다. CNI Agent가 `Running`이어도 개별 Pod의 구성 성공이나 모든 목적지의 도달성을 대신 증명하지는 않는다.

### 다른 Node의 Pod로만 통신이 안 되는 경우

4주차에서 본 **Overlay와 Underlay를 함께 확인**한다.

| 구성 | 주요 점검 대상 |
| --- | --- |
| **Flannel VXLAN** | 노드 IP 도달성, VXLAN 장치·전달 정보, 설정된 UDP 포트 |
| **Calico 비캡슐화 / BGP 사용** | BGP 세션, 원격 Pod 대역 경로, 기반망의 반환 경로 |
| **IP-in-IP** | IP 프로토콜 4 허용, 터널·경로·MTU |
| **Cilium** | 터널 또는 Native Routing 구성, Agent 상태, 정책·drop 사유 |

**Node IP로 ping이 성공해도 VXLAN UDP나 IPIP가 허용된다고 판단할 수는 없다.**

문제 Node 호스트에서 확인하는 예시:

```bash
ip -br addr
ip -4 route
ip rule
ip -4 route get <remote-pod-ip>

# 일반적인 Flannel VXLAN 구성일 때
ip -d link show flannel.1
bridge fdb show dev flannel.1
ip neigh show dev flannel.1

# Calico에서 BGP를 사용하는 구성일 때
sudo calicoctl node status
```

`ip route get`은 실행한 네트워크 namespace의 조회 결과이다. Pod 출발지에 따른 정책 라우팅이 있으면 Pod 내부의 경로와 호스트의 `ip rule`도 함께 확인한다.

### 실제 사례 1. DNS 오류로 시작했지만 원인은 Flannel UDP 차단

- **환경:** 2018년, AWS / Kubernetes 1.9.2 / CentOS 7.4 / Flannel 0.9.1
- **증상:** 외부 이름 해석 실패와 다른 Node의 Pod 접속 실패
- **핵심 단서:** BusyBox가 DNS Pod와 같은 Node에 있을 때만 DNS 질의 성공
- **작성자의 조사:** DNS 설정 등을 점검한 뒤 UDP 통신이 차단되어 있음을 발견
- **보고된 해결:** UDP 허용 후 통신 정상화

```
같은 Node의 DNS Pod → 성공
다른 Node의 DNS Pod → 실패
다른 Node의 일반 Pod → 실패
                 ↓
DNS 프로세스뿐 아니라 노드 간 전달 경로 조사
```

**이름 해석 실패라는 증상과, DNS 패킷이 목적지까지 도달하지 못한 원인을 구분하는 사례**

원문은 허용한 정확한 UDP 포트를 밝히지 않는다. Linux Flannel VXLAN의 기본값은 현재 공식 문서에서 UDP `8472`로 설명하지만, 실제 포트는 해당 구성에서 확인한다. VXLAN과 별도 `UDP` backend를 혼동하지 않는다.

#### 같은 유형을 조사한다면

문제 Node 양쪽에서 실제 요청과 함께 관찰한다. 아래는 Linux Flannel VXLAN 기본 포트를 가정한 학습용 명령이다.

```bash
# Node의 기반망 NIC에서: 캡슐화된 트래픽
sudo tcpdump -ni <underlay-interface> -nn -c 30 'udp port 8472'

# 터널 인터페이스에서: 역캡슐화된 내부 Pod 트래픽
sudo tcpdump -ni flannel.1 -nn -c 30 'host <remote-pod-ip>'
```

| 관찰 | 다음 조사 |
| --- | --- |
| 송신 Node에서 보이지만 수신 Node에서 보이지 않음 | 두 Node 사이 경로·방화벽·클라우드 네트워크 규칙 |
| 수신 Node의 기반망 NIC에는 도착 | 터널 설정·역캡슐화·정책·Pod 전달 |
| 요청은 목적지에 도착하지만 응답이 돌아오지 않음 | 앱 응답, 반대 방향 경로·정책 |

캡처가 없다는 결과는 **선택한 인터페이스·필터·관찰 시간에서 보이지 않았다는 의미**이다. 다른 경로를 사용했는지도 확인한다.

---

## Pod-to-Service / kube-proxy 장애

**Service IP는 실제 backend Pod와 연결하여 제공하는 주소이다. Service 설정과 backend 통신을 분리해 확인한다.**

### Service → EndpointSlice → Pod

```
Service: 10.96.0.100:80
  selector: app=api
        ↓
EndpointSlice: backend 주소·포트·상태
        ↓
10.244.1.20:8080 / 10.244.2.20:8080
```

```bash
kubectl -n netlab get svc api -o yaml
kubectl -n netlab get pods -l app=api -o wide --show-labels
kubectl -n netlab get endpointslices \
  -l kubernetes.io/service-name=api -o yaml
```

| 확인 대상 | 확인 내용 |
| --- | --- |
| **selector** | 실제 backend Pod label과 일치하는가 |
| **port / targetPort / protocol** | Service 포트와 실제 앱 리스닝 포트를 올바르게 연결하는가 |
| **EndpointSlice** | 기대한 주소·포트가 있고, 준비·종료 상태가 적절한가 |
| **readiness** | 앱이 준비되지 않아 일반적인 트래픽 대상에서 제외된 것은 아닌가 |
| **backend 직접 요청** | 실제 backend들이 같은 출발지에서 각각 응답하는가 |

이 절차는 **selector가 있는 일반 ClusterIP Service**를 기준으로 한다. Headless Service와 수동으로 EndpointSlice를 관리하는 구성은 해당 방식에 맞춰 해석한다.

### Service를 구현하는 구성 요소 확인

| 환경 | 제어·관리 | 실제 트래픽 처리 |
| --- | --- | --- |
| **kube-proxy** | Service·EndpointSlice 감시 후 규칙 갱신 | 커널의 iptables·nftables·IPVS 등 해당 모드 |
| **Cilium kube-proxy replacement** | Agent가 Service·backend 정보 관리 | 구성에 따른 socket·패킷 Hook의 eBPF 프로그램 |

**kube-proxy가 모든 요청을 사용자 공간에서 직접 중계하는 것은 아니다.**

```bash
# kube-proxy를 사용하는 일반적인 배포
kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide
kubectl -n kube-system logs <problem-node-kube-proxy-pod> \
  --since=15m --timestamps

# Node 호스트: 실제 사용 모드에 해당하는 도구만 선택
sudo iptables-save -t nat
sudo nft list ruleset
sudo ipvsadm -Ln
```

- API 접근 오류와 규칙 동기화 실패 확인
- 정상 Node와 실패 Node의 Service·backend 반영 결과 비교
- 방화벽 재적용이나 다른 관리 도구가 규칙에 영향을 주었는지 확인
- `KUBE-*` iptables 체인이 없다는 이유만으로 장애로 단정하지 않음
- Cilium replacement 환경은 아래 Cilium 진단 절차로 확인

---

## DNS 장애

**이름 해석 단계와, 해석된 IP로 실제 요청을 보내는 단계를 구분한다.**

```
이름 사용
  → resolver의 DNS 질의
  → 구성된 DNS 서버
  → IP 획득
  → 해당 IP·포트로 실제 연결
```

### 기본 점검

```bash
# 출발 Pod의 resolver 설정과 질의 결과
kubectl -n netlab exec client -- cat /etc/resolv.conf
kubectl -n netlab exec client -- nslookup kubernetes.default.svc.cluster.local
kubectl -n netlab exec client -- nslookup api.netlab.svc.cluster.local

# 일반적인 CoreDNS 배포: 실제 이름·label은 환경에서 확인
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
kubectl -n kube-system get svc kube-dns
kubectl -n kube-system get endpointslices \
  -l kubernetes.io/service-name=kube-dns -o yaml
kubectl -n kube-system logs <coredns-pod> --since=15m --timestamps
```

NodeLocal DNSCache를 사용하면 Pod의 nameserver가 로컬 캐시 주소일 수 있다. 실제 resolver → 캐시 → CoreDNS 경로를 확인한다.

| 결과 | 점검 방향 |
| --- | --- |
| **NXDOMAIN** | 이름·Namespace·Service 존재 여부 확인 |
| **SERVFAIL** | CoreDNS 설정·권한·상위 DNS 오류 확인 |
| **timeout** | DNS 서버까지의 경로·정책·UDP/TCP·서버 부하 확인 |
| **내부 이름만 성공** | 외부 질의를 전달하는 상위 DNS 경로 확인 |
| **이름은 올바르게 해석되지만 연결 실패** | 해석 이후의 Service·Ingress·외부 경로 확인 |

timeout만으로 CoreDNS 프로세스의 문제를 확정하지 않는다. 사례 1처럼 노드 간 경로 장애일 수도 있다.

```bash
# dig가 있는 진단 환경: 실제 nameserver로 변경
dig @<dns-server-ip> api.netlab.svc.cluster.local A +time=2 +tries=1
dig @<dns-server-ip> api.netlab.svc.cluster.local A +tcp +time=2 +tries=1
```

DNS는 UDP와 TCP를 모두 사용한다. NetworkPolicy의 허용 대상·포트를 확인하고, 응답이 없는 경우 요청·응답 양쪽의 관찰 결과를 비교한다.

---

## NetworkPolicy 장애

**Pod의 label·Namespace·IP·포트 등을 기준으로 허용할 통신을 정의하는 정책**

| 확인 사항 | 의미 |
| --- | --- |
| **정책 집행 지원** | 지원하는 네트워크 구현이 있어야 실제 적용 |
| **송신 측 egress** | 출발 Pod에서 나가는 연결이 허용되는가 |
| **수신 측 ingress** | 목적지 Pod로 들어오는 연결이 허용되는가 |
| **selector** | 현재 Pod·Namespace의 실제 label과 일치하는가 |
| **규칙 합산** | 기본 Kubernetes NetworkPolicy의 허용 규칙은 합산됨 |
| **DNS 의존성** | 앱 목적지뿐 아니라 이름 해석 경로도 필요한가 |

`podSelector`만 사용하면 해당 정책과 같은 Namespace의 Pod를 선택한다. 다른 Namespace의 특정 Pod를 선택하려면 같은 `from` 또는 `to` 항목 안에 Namespace와 Pod 조건을 함께 지정한다.

```bash
kubectl -n <namespace> get networkpolicy -o yaml
kubectl -n <namespace> describe networkpolicy <policy>
kubectl -n <namespace> get pods --show-labels
kubectl get namespaces --show-labels
```

CiliumNetworkPolicy·Calico GlobalNetworkPolicy 등 추가 정책이 있는 환경에서는 그 정책도 함께 확인한다. 위의 합산 설명은 **기본 Kubernetes NetworkPolicy** 기준이다.

### 실제 사례 2. port-forward는 되지만 Traefik 접속은 Gateway Timeout

- **환경:** 2020년, Traefik / Flask 애플리케이션
- **증상:** 도메인과 Ingress 경로로 요청하면 Gateway Timeout
- **비교:** `kubectl port-forward`로는 애플리케이션 정상 응답
- **작성자가 설명한 원인:** 다른 Namespace의 Traefik에서 들어오는 통신 허용 누락
- **보고된 해결:** Namespace 간 ingress 허용 정책 추가

```
port-forward → 대상 앱                         성공

클라이언트 → Traefik → 대상 앱                 실패
                        ↑
             이 구간의 정책·경로 확인
```

**port-forward 성공은 앱이 응답할 수 있다는 단서이지, 실제 Ingress 경로가 정상이라는 증거는 아니다.**

원문의 해결 정책은 모든 Namespace에서 대상 Namespace의 모든 Pod로 들어오는 트래픽을 넓게 허용한다. 이 변경으로 복구했다고 보고했지만, 운영에서는 실제 Controller와 앱·포트 범위로 좁혀 검토한다.

### 필요한 출발지만 허용하는 설계 예시

아래는 원문 정책을 그대로 복사한 것이 아니라, **범위를 제한하는 학습용 예시**이다. 실제 label과 앱 포트를 먼저 확인한다.

- 대상: `netlab` Namespace의 `app=api` Pod
- 출발: `ingress-system` Namespace의 `app.kubernetes.io/name=traefik` Pod
- 허용: TCP `8080`
- 출발 측 egress가 격리되어 있다면 그쪽의 허용도 필요

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-traefik-to-api
  namespace: netlab
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-system
          podSelector:
            matchLabels:
              app.kubernetes.io/name: traefik
      ports:
        - protocol: TCP
          port: 8080
```

`namespaceSelector`와 `podSelector`는 **같은 항목이므로 AND 조건**이다. 다른 정책이 추가로 허용한 통신은 이 정책 하나로 취소되지 않는다.

---

## IPAM / IP Pool 장애

**Pod IP의 할당·회수 상태와, 해당 Pod가 실제로 사용할 수 있는 주소 범위를 확인하는 과정**

### 기존 Pod는 정상인데 새 Pod만 생성되지 않는 경우

```
새 Pod에 대한 CNI ADD
       ↓
IPAM에 주소 요청
       ↓
사용 가능한 주소·블록·클라우드 자원 확보 실패
       ↓
네트워크 구성 실패 / sandbox 생성 오류
```

다음은 IPAM 오류의 **문구 예시**이다. 모든 CNI가 같은 메시지를 출력하는 것은 아니다.

```
failed to assign an IP address to container
no IP addresses available
InsufficientCidrBlocks
```

| 점검 대상 | 확인할 것 |
| --- | --- |
| **전체 Pool / 서브넷** | 사용량·잔여량, 확장 가능한 주소 범위 |
| **할당 대상** | disabled 여부, nodeSelector·명시적 Pool 선택 |
| **노드별 블록·한도** | 해당 Node가 사용할 수 있는 블록·IP·ENI 등의 한도 |
| **회수 상태** | 삭제된 워크로드와 IPAM 할당 기록의 불일치 |
| **주소 중복·겹침** | Pod·Service·Node·VPN·추가 네트워크 대역의 충돌 |

Calico IPAM을 사용하는 환경의 예시:

```bash
# 클러스터에 맞는 calicoctl 버전·접속 설정 필요
calicoctl get ippool -o yaml
calicoctl ipam show
calicoctl ipam show --show-blocks
```

전체 여유량과 블록별 상태를 함께 확인한다. Pool의 `disabled`·`nodeSelector` 등에 따라 여유 IP가 있어도 특정 Node의 할당 대상이 아닐 수 있다.

### EKS Prefix Delegation: 빈 IP 개수와 연속 공간은 다르다

AWS 공식 문서에 설명된 **환경별 실패 유형**이다. 앞의 커뮤니티 사례와는 별개의 참고 내용이다.

- VPC CNI Prefix Delegation에서 IPv4 prefix를 할당하려면 연속된 `/28` 공간 필요
- 빈 IP가 여러 개 있어도 흩어져 있으면 prefix 할당 실패 가능
- CNI 로그에서 `InsufficientCidrBlocks` 등 실제 오류 확인
- 신규 서브넷·prefix 예약 등 주소 공간 설계를 검토

**IP 개수 부족과 주소 단편화를 구분해야 한다.**

### 6주차 Multi-Networking과 연결

`host-local`은 한 호스트 안의 할당 상태를 관리한다. 같은 L2망에서 여러 Node가 같은 할당 범위를 독립적으로 사용하면 IP 중복을 막지 못할 수 있다.

- 노드별 범위를 분리하거나, 네트워크 요구에 맞는 공유 IPAM 사용
- 실제 사용 중인 주소인지 확인한 뒤 해당 IPAM의 회수 절차 사용
- IPAM 기록 파일 삭제나 강제 IP 회수를 기본 해결책으로 사용하지 않음

---

## External 통신 장애

### Pod → 외부 시스템

**목적지까지 가는 경로와, 응답이 Pod로 돌아오는 경로를 함께 확인한다.**

```
Pod
 → 목적지 이름 해석
 → egress 정책·라우팅
 → 환경에 따라 Node/Egress Gateway의 SNAT
 → 필요하면 NAT Gateway·외부 라우터
 → 외부 시스템
 ← 반환 경로와 연결 상태에 따른 응답 전달
```

외부에 Pod 대역으로 돌아오는 경로가 없다면 SNAT 등이 필요할 수 있다. Pod IP가 외부에서도 라우팅되는 환경까지 모두 같은 NAT 구조로 설명하지 않는다.

| 관찰 | 확인할 것 |
| --- | --- |
| Node에서는 성공, Pod에서는 실패 | Pod 정책·경로·NAT·출발지 차이 |
| 이름 해석부터 실패 | DNS / DNS egress |
| DNS는 정상, 연결 timeout | 목적지 포트·외부 방화벽·egress·NAT·반환 경로 |
| 특정 외부 목적지만 실패 | 목적지 측 허용 IP·포트, 경로·대역 충돌 |
| TLS 단계에서 실패 | SNI·인증서·시간·중간 프록시와 MTU 등 실제 오류 |

6주차처럼 `net1`을 추가했다면 **실제로 어떤 인터페이스와 출발지 IP가 선택되는지**도 확인한다.

```bash
# Pod 내부: 실제 외부 목적지 IP로 변경
kubectl -n <namespace> exec <client-pod> -- ip -4 route get <external-ip>
kubectl -n <namespace> exec <client-pod> -- ip rule
```

`eth0`가 Primary Network라는 사실과, 특정 목적지의 선택 경로는 별개이다. 추가 default route·정책 라우팅·앱 바인딩을 함께 확인한다.

### External → Service / Ingress

```
외부 클라이언트
 → DNS / LoadBalancer / NodePort
 → Ingress Controller
 → backend 연결
 → 애플리케이션
```

- LB가 실제로 어느 Node에 트래픽을 전달하는지 확인
- Controller의 배치·로그·라우팅 설정 확인
- Service의 backend·포트·준비 상태 확인
- Controller에서 앱으로 가는 정책 확인: 사례 2
- `externalTrafficPolicy`와 로컬 endpoint·LB health check 확인

**Gateway Timeout만으로 backend 앱의 장애라고 단정하지 않는다. Controller에서 backend까지 연결이 가능한지 확인한다.**

### 실제 사례 3. 업그레이드 후 특정 Node에서 외부 도메인 접속 실패

| 항목 | 작성자가 보고한 내용 |
| --- | --- |
| **시점·환경** | 2024년 3월, Kubernetes 1.23 → 1.26, MetalLB·Traefik |
| **성공 경로** | 내부 Service 이름으로 접속 |
| **실패 경로** | 외부용 도메인으로 접속할 때 일부 요청 timeout |
| **DNS** | 올바른 IP로 해석됨 |
| **Node 차이** | 출발 앱과 같은 Node에 Traefik Pod가 있을 때 성공 |
| **보고된 조치** | Traefik Service의 `externalTrafficPolicy: Local` 제거 후 정상화 |

**단순한 “간헐적 실패”를 Pod 배치와 연결해 성공·실패 조건으로 좁힌 사례**

#### 공식 동작과 구분할 부분

- `externalTrafficPolicy: Local`은 외부 트래픽을 로컬 endpoint로 전달하는 정책이다.
- 현재 API 문서는 클러스터 내부에서 External IP·LoadBalancer IP로 보내는 트래픽은 `Cluster` 의미를 적용한다고 명시한다.
- 따라서 게시물의 관찰을 모든 환경에서의 정상 동작으로 일반화할 수는 없다. 실제 NAT·프록시 규칙·트래픽 경로는 추가 확인 대상이다.
- 작성자가 “1.26 안정화”의 근거로 연결한 이슈는 `externalTrafficPolicy`가 아니라 `internalTrafficPolicy` 이슈이다.

확인된 범위는 **해당 작성자가 설정 변경 후 복구를 보고했다는 것**이다. 업그레이드가 장애를 만든 정확한 메커니즘까지 확인된 사례로 인용하지 않는다.

#### 같은 유형의 점검 예시

```bash
kubectl -n <ingress-namespace> get svc <traefik-service> -o yaml
kubectl -n <ingress-namespace> get pods -o wide
kubectl -n <ingress-namespace> get endpointslices \
  -l kubernetes.io/service-name=<traefik-service> -o yaml
```

출발 Node·실제 유입 Node·Controller 배치·정책 값을 대조한다. 정책을 바꾼다면 클라이언트 IP 보존과 LB 동작에 미치는 영향도 검증한다.

---

## MTU / conntrack: 간헐적 장애 분석

### MTU 불일치

**경로에서 전달할 수 있는 크기보다 큰 패킷이 만들어지면, 단편화·PMTUD·재전송 동작에 따라 큰 전송만 실패할 수 있다.**

4주차의 IPv4 VXLAN 예시:

```
기반 경로 MTU 1500
  - 추가 Ethernet / IPv4 / UDP / VXLAN 오버헤드 50
  = 내부 IP 패킷 MTU 1450
```

이는 일반적인 IPv4 VXLAN 조건이다. IPv6·다른 터널·암호화·중첩 네트워크에서는 값이 달라진다.

```bash
# Node 또는 진단 Pod의 실제 인터페이스 확인
ip -d link show

# Linux iputils ping, IPv4 / 실제 ICMP 허용 대상 기준
ping -4 -M do -s 1422 -c 3 <destination-ip>
```

위 ping의 `1422`는 payload이며 IPv4 20 + ICMP 8을 더하면 1450바이트이다. 값과 성공 여부를 비교하되, ICMP 차단 가능성 때문에 실패만으로 MTU 문제를 확정하지 않는다.

- 작은 요청과 큰 응답·파일 전송 비교
- 같은 Node와 다른 Node 비교
- 경로 MTU와 터널·Pod 인터페이스 MTU 비교
- 재전송, ICMP fragmentation-needed 등 확인
- MTU 변경 후 기존 workload에도 반영되었는지 확인

### conntrack 자원·상태 문제

**커널이 연결과 NAT 상태를 추적하는 기능. 테이블 용량 문제와 TCP 상태 판정 문제를 나누어 확인한다.**

Node 호스트의 Linux conntrack을 사용하는 경로에서 확인한다.

```bash
sysctl net.netfilter.nf_conntrack_count
sysctl net.netfilter.nf_conntrack_max
sudo conntrack -S
sudo journalctl -k --since "15 min ago"
```

| 관찰 | 해석 |
| --- | --- |
| count가 max에 접근 | 용량 압박의 단서 |
| table full·drop·insert 실패 증가 | 연결 추적 자원 문제 추가 조사 |
| RST·INVALID 증가 | TCP 상태·패킷 순서·커널·Service 구현 조사 |
| 특정 부하에서만 실패 | 연결 생성률·유지 시간·트래픽 분산과 대조 |

`count`만으로 원인을 확정하지 않는다. 용량 확대가 필요하다면 메모리 비용과 실제 연결 패턴도 검토한다. eBPF 기반 구현에서는 해당 구현의 연결 추적 Map도 별도 확인한다.

공식 Service 문서는 커널과 kube-proxy 모드에 따른 conntrack 우회 처리 차이도 설명한다. 과거 사례의 sysctl·규칙을 버전 확인 없이 적용하지 않는다.

---

## Cilium / Hubble로 확인하기

**5·7주차에서 배운 Agent·eBPF Map·관찰 이벤트를, 실패한 통신의 출발지·목적지와 연결한다.**

### Agent와 Service 정보

관리용 터미널에서 문제 Node의 Cilium Pod를 선택한다. 아래는 `kube-system` 배포 예시이다.

```bash
kubectl -n kube-system get pods -l k8s-app=cilium -o wide

kubectl -n kube-system exec <cilium-pod> -c cilium-agent -- \
  cilium-dbg status --verbose

kubectl -n kube-system exec <cilium-pod> -c cilium-agent -- \
  cilium-dbg service list

kubectl -n kube-system logs <cilium-pod> -c cilium-agent \
  --since=15m --timestamps
```

- Agent의 API 연결·Controller·IPAM 상태 확인
- 실제 Service frontend와 backend 정보 대조
- 실패 Node와 정상 Node를 비교
- 사용하는 Cilium 버전의 명령·Datapath 모드 확인

### Hubble Flow 확인

Hubble CLI와 Relay 연결이 준비된 환경에서 실행한다.

```bash
hubble status
hubble observe --pod netlab/client --since 5m
hubble observe --pod netlab/client --since 5m --verdict DROPPED
```

| 확인할 정보 | 판단에 사용하는 방법 |
| --- | --- |
| **출발지·목적지** | 조사 중인 요청과 같은 흐름인지 확인 |
| **verdict·drop 사유** | 정책 거부 등 관찰된 원인 확인 |
| **방향·포트·프로토콜** | 예상한 통신과 일치하는지 비교 |
| **요청 시각** | 앱 오류·다른 Node의 로그와 대조 |

**DROPPED의 사유를 확인해야 정책 차단과 다른 drop을 구분할 수 있다.** FORWARDED 이벤트 하나가 외부 시스템의 최종 응답 성공까지 증명하지는 않는다. 이벤트가 없으면 조회 대상·관찰 지점·보존 시간도 확인한다.

---

## Operations

### 장애 유형별 대응 방향

| 유형 | 복구 방향 | 재발 방지 |
| --- | --- | --- |
| **Service selector·포트 오류** | 실제 Pod·포트에 맞게 수정 | 배포 후 Service 경유 요청 확인 |
| **노드 간 경로·터널 차단** | 실제 전달 방식의 경로·필요 통신 복구 | Node 간 Pod 통신 검사, 기반망 규칙 관리 |
| **정책 누락** | 필요한 출발지·목적지·포트 허용 | DNS·Ingress·의존 서비스 통신 함께 검토 |
| **IPAM 부족·제약** | Pool·서브넷·블록·노드 한도에 맞게 조치 | 잔여량·예약량·노드별 편중 관측 |
| **외부 NAT·반환 경로 문제** | 실제 egress 경로와 주소 변환 복구 | 외부 의존성·SNAT/NAT 자원 관측 |
| **MTU 문제** | 실제 경로와 캡슐화에 맞게 조정 | 큰 전송을 포함한 경로 검증 |
| **conntrack 문제** | 용량·트래픽 패턴·커널 원인별 조치 | 사용량·실패 통계·재전송 추세 관측 |

### 복구 후 확인

1. **최초 실패한 출발지·목적지·프로토콜로 재검증**
2. 같은 Node / 다른 Node / 여러 backend 결과 비교
3. Pod IP / Service IP / DNS 이름 / 외부 경로 중 영향받은 구간 확인
4. 원래 문제가 발생한 동시성·응답 크기·연결 유지 조건으로 확인
5. 새 연결과 기존 연결을 구분하고, 새 Pod 네트워크 구성도 확인
6. 오류율·지연·drop이 정상 범위로 돌아왔는지 관찰

**“재시작 후 한 번 성공”보다, 원래 실패 조건에서 반복해서 정상인지 확인하는 것이 중요하다.**

### 장애 기록

| 항목 | 기록할 내용 |
| --- | --- |
| **영향** | 언제부터 어떤 요청이 어느 범위에서 실패했는가 |
| **관찰** | 성공·실패 경로, Node·Pod 배치, 로그·캡처 시각 |
| **원인** | 확인된 원인과 아직 검증하지 못한 가설 |
| **조치** | 변경한 리소스·설정과 복구 확인 방법 |
| **후속 작업** | 모니터링·주소 계획·정책·배포 절차 개선 |

---

## 참고자료

- [Stack Overflow: Flannel 노드 간 통신과 UDP 차단](https://stackoverflow.com/questions/52799982/kuberentes-communication-is-not-working-cross-nodes)
- [Stack Overflow: Traefik 경유 Gateway Timeout과 NetworkPolicy](https://stackoverflow.com/questions/61275109/problem-using-traefik-as-load-balancer-in-kubernetes)
- [Reddit: Kubernetes 1.26 업그레이드 후 접속 실패](https://www.reddit.com/r/kubernetes/comments/1bdo1rz/networking_issue_since_upgrade_to_kubernetes_126/)
- [Kubernetes](https://kubernetes.io/docs/)
- [Flannel](https://github.com/flannel-io/flannel)
- [Calico](https://docs.tigera.io/calico/latest/)
- [CNI](https://www.cni.dev/)
- [AWS EKS](https://docs.aws.amazon.com/eks/)
- [Cilium](https://docs.cilium.io/en/stable/)
