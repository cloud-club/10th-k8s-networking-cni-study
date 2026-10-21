# Week 8. Kubernetes Networking Common Issue Troubleshooting & Operations

# 장애 분석 기준

## 통신 경로별 담당 컴포넌트

| 통신 경로 | 지나가는 구간 | 담당 |
| --- | --- | --- |
| Pod → Pod (같은 노드) | veth → bridge 또는 eBPF → veth | CNI |
| Pod → Pod (다른 노드) | veth → 노드 라우팅 / Overlay → 상대 노드 | CNI, 노드 라우팅, 클라우드 네트워크 |
| Pod → Service | ClusterIP → DNAT → backend Pod | kube-proxy, EndpointSlice |
| Pod → Service 이름 | resolv.conf → CoreDNS → ClusterIP | CoreDNS |
| Pod → 외부 | 노드에서 SNAT → NAT GW / IGW | CNI masquerade, 클라우드 라우팅 |
| 외부 → Pod | LB / NodePort → kube-proxy → Pod | LB 컨트롤러, kube-proxy |
- Pod 네트워크 구성은 CNI, Service 트래픽은 kube-proxy 담당이다.
- Cilium kube-proxy replacement 환경이면 Service 처리도 CNI 쪽으로 넘어온다

## 범위 좁히기

- IP로는 되는데 이름으로 안 됨
    
    → DNS 문제
    
- Pod IP로는 되는데 ClusterIP로 안 됨
    
    → Endpoint나 kube-proxy 문제
    
- 같은 노드 Pod끼리는 되는데 다른 노드 Pod로 안 됨
    
    → 노드 간 라우팅, Overlay, MTU 문제
    
- 특정 Pod만 안 됨
    
    → NetworkPolicy나 그 Pod 자체 문제일 가능성이 크다
    
- 클러스터 전체가 안 됨
    
    → CNI, CoreDNS, 노드 레벨 문제부터 봐야 한다
    

# 자주 발생하는 장애 사례

## CNI / IPAM

### 1. Pod가 ContainerCreating에서 멈춤

- 증상
    - Pod Events에 `failed to create pod sandbox`가 찍힌다
    - 노드가 NotReady이고 `NetworkPluginNotReady`, `cni plugin not initialized`가 보이기도 한다
- 원인
    - CNI DaemonSet Pod(calico-node, cilium, aws-node 등) 장애
    - `/etc/cni/net.d/` 설정 파일이나 `/opt/cni/bin/` 바이너리 누락
- 확인
    - `kubectl describe pod`, CNI DaemonSet Pod 상태와 로그
- 대응
    - CNI Pod 복구, 설정과 바이너리 재배포
- 이미 떠 있던 Pod는 통신이 유지되는 경우도 있어서 새 배포를 하고 나서야 알아채기도 한다

### 2. 새 Pod만 IP를 못 받음

- 증상
    - 기존 Pod는 멀쩡한데 새로 뜨는 Pod만 ContainerCreating에서 멈춘다
- 원인
    - IP Pool 또는 서브넷 소진
    - 노드마다 블록을 나눠 주는 방식(Calico IPAM block, 노드별 podCIDR)이면 전체 Pool은 남아 있어도 특정 노드 블록만 꽉 찰 수 있다
    - AWS VPC CNI는 인스턴스 타입별 ENI 수와 ENI당 IP 수가 노드당 최대 Pod 수를 정한다
    - Pod 비정상 종료 후 IP가 반납되지 않는 누수
- 확인
    - `calicoctl ipam show`, AWS라면 ipamd 로그와 서브넷 가용 IP 수
- 대응
    - Pool 확장, 서브넷 추가, 누수 IP 정리
    - AWS VPC CNI라면 prefix delegation도 방법이다

### 3. CNI 교체나 업그레이드 후 일부 노드만 이상함

- 원인
    - `/etc/cni/net.d/`에 설정 파일이 여러 개 있으면 이름순 첫 번째 파일을 쓴다
    - 예전 CNI 설정 파일, 인터페이스, iptables 규칙이 남아서 엉뚱한 플러그인이 동작한다
- 확인
    - `/etc/cni/net.d/` 파일 목록, `ip link`
- 대응
    - 남은 설정 정리 후 노드 재시작, Pod 재생성

## Service / kube-proxy

### 4. Service 호출이 안 됨

- 증상
    - connection refused 또는 응답 없음
- 원인
    - Endpoint가 비어 있음
        - Service selector와 Pod label 불일치
        - Pod가 readiness probe를 통과하지 못함
    - Service `targetPort`가 컨테이너 포트와 다름
- 확인
    - EndpointSlice, Pod label, Service 포트 정의
- 대응
    - label과 포트 수정, readiness probe 점검
- 운영에서 제일 자주 보는 케이스다. 네트워크 문제처럼 보이는데 실제론 설정 문제인 경우가 많다

### 5. Pod IP는 되는데 ClusterIP만 안 됨

- 원인
    - kube-proxy가 죽었거나 API 서버와 끊겨서 Service 규칙이 갱신되지 않음
    - 새 Pod로는 트래픽이 안 가고, 이미 없어진 Pod로 트래픽이 간다
- 확인
    - kube-proxy 로그
    - iptables 모드면 `KUBE-SERVICES` → `KUBE-SVC-*` → `KUBE-SEP-*` 순서로 규칙 추적
    - ipvs 모드면 `ipvsadm -Ln`에서 backend가 붙어 있는지 확인
- 대응
    - kube-proxy 재시작, 모드와 설정 점검

### 6. 트래픽 많을 때 새 연결이 간헐적으로 실패

- 원인
    - conntrack 테이블이 가득 차서 새 연결이 버려짐
- 확인
    - `dmesg`에 `nf_conntrack: table full, dropping packet`
    - `conntrack -C`와 `nf_conntrack_max` 비교
- 대응
    - `nf_conntrack_max` 상향, 커넥션 재사용

## DNS

### 7. 이름 해석 실패 또는 간헐적인 5초 지연

- 원인
    - CoreDNS Pod 수 부족, 메모리 부족으로 인한 재시작
    - Pod resolv.conf 기본값 `ndots:5` 때문에 외부 도메인도 search 도메인을 먼저 붙여 질의하고, 실패 질의가 여러 번 나가서 부하와 지연이 커진다
    - A와 AAAA를 같은 소켓으로 동시에 보내다가 conntrack 경합으로 패킷 하나가 버려지면 5초 타임아웃이 난다
- 확인
    - Pod resolv.conf의 nameserver가 kube-dns ClusterIP인지
    - FQDN과 짧은 이름으로 각각 nslookup
    - CoreDNS 로그와 메트릭
- 대응
    - CoreDNS 증설, NodeLocal DNSCache 도입
    - 외부 도메인 호출이 많으면 Pod `dnsConfig`로 ndots 낮추기
    - 이름 끝에 점을 붙여 FQDN으로 쓰기

## NetworkPolicy

### 8. 정책 적용 후 통신이 다 끊김

- 원인
    - default deny egress를 걸면서 DNS(UDP/TCP 53)까지 같이 막음
    - `from` 아래 namespaceSelector와 podSelector를 한 항목으로 쓰면 AND, 별도 항목으로 쓰면 OR인데 이걸 헷갈림. YAML에서  하나 차이다
- 확인
    - 대상 Pod를 고르는 정책 목록
    - Cilium이면 Hubble DROPPED와 drop reason
- 대응
    - kube-system CoreDNS로 가는 53 허용 정책 추가, selector 구조 다시 확인
- 정책이 하나도 없으면 전부 허용이라, 처음엔 잘 되다가 정책 하나 추가하자마자 막히는 경우가 많다

### 9. 정책을 만들었는데 적용이 안 됨

- 원인
    - NetworkPolicy는 CNI가 구현해야 동작한다. Flannel만 쓰는 클러스터는 정책을 만들어도 무시된다
- 대응
    - 정책을 지원하는 CNI로 교체하거나 Calico 같은 정책 엔진을 함께 쓴다

## Routing / Overlay

### 10. 다른 노드 Pod로만 통신이 안 됨

- 원인
    - 노드 간 라우팅 누락, BGP 피어 다운
    - 노드 사이 보안그룹이나 방화벽에서 Overlay 포트 차단
        - VXLAN: UDP 8472(Flannel, Cilium 기본) 또는 4789(Calico VXLAN)
        - Geneve: UDP 6081
        - BGP: TCP 179
    - 클라우드에서 Overlay 없이 Pod IP를 바로 라우팅하는데 Source/Destination Check가 켜져 있거나 VPC 라우팅 테이블에 Pod CIDR이 없음
- 확인
    - `ip route`, `calicoctl node status`, 보안그룹 규칙
    - 출발 노드와 도착 노드 양쪽에서 tcpdump를 떠서 어디서 사라지는지 확인
- 대응
    - 포트 허용, 라우팅 추가, BGP 복구, Source/Destination Check 해제

### 11. 작은 요청은 되는데 큰 응답만 멈춤

- 원인
    - MTU 불일치. Overlay 헤더만큼 실제 보낼 수 있는 크기가 줄어든다
- 확인
    - 노드 NIC와 Pod 인터페이스 MTU 비교
    - `ping -M do -s <size> <ip>`로 단편화 없이 보낼 수 있는 크기 확인
- 대응
    - CNI MTU를 노드 MTU에서 캡슐화 헤더만큼 뺀 값으로 맞춘다
- 연결도 되고 헤더까지 오는데 본문에서 멈춰서 처음엔 애플리케이션 문제로 오해하기 쉽다

## External

### 12. Pod에서 외부로 못 나감

- 원인
    - masquerade(SNAT) 설정 누락, NAT GW나 라우팅, 보안그룹 문제
- 확인
    - 같은 노드에서 노드 자체는 외부로 나가지는지 비교
    - `iptables -t nat -L`로 masquerade 규칙 확인
- 대응
    - masquerade 설정, NAT GW와 egress 규칙 점검

### 13. LoadBalancer Service가 Pending

- 원인
    - Cloud Controller나 MetalLB 같은 LB 구현체가 없음
- 확인
    - `kubectl describe svc`의 Events
- 대응
    - LB 컨트롤러 설치, 권한 확인

### 14. 외부 요청이 일부 노드로 갈 때만 실패

- 원인
    - `externalTrafficPolicy: Local`인데 그 노드에 backend Pod가 없음
- 확인
    - Service 설정, Pod 분포, LB 헬스체크 상태
- 대응
    - LB가 헬스체크 포트를 보고 있는지 확인, Pod 분산 또는 Cluster로 변경
- Local은 클라이언트 IP를 보존하려고 쓰는데, 대신 Pod 없는 노드는 트래픽을 버린다

# 정리

- 실제로 자주 보이는 건 Endpoint 없음, DNS, 정책 때문에 DNS 막힘, IP 소진, MTU 정도다.
- 다만 이건 일반적으로 많이 언급되는 경향이고 공식 통계는 아니다. 환경마다 다름
- Cilium 환경이면 Hubble DROPPED와 drop reason으로 원인 영역을 가장 빨리 좁힐 수 있다

---

- 참고 자료
    - [Kubernetes – Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
    - [Kubernetes – Debugging DNS Resolution](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)
    - [Kubernetes – Network Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
    - [Kubernetes – Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
    - [Kubernetes – Using NodeLocal DNSCache](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/)
    - [Kubernetes – DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
    - [Cilium – Troubleshooting](https://docs.cilium.io/en/stable/operations/troubleshooting/)