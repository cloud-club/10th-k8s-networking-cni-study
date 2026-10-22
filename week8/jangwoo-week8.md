# Week 8. Kubernetes Network 장애 분석 핵심

네트워크 장애는 구성요소를 하나씩 외우기보다 **Packet이 지나가는 경로를 구간별로 나눠서** 확인한다.

```text
Application
  → DNS
  → Service
  → NetworkPolicy
  → CNI / Routing
  → 목적지 Pod 또는 외부 Network
  → Reply 경로
```

앞 구간이 정상임을 확인한 뒤 다음 구간으로 이동해야 원인 범위를 빠르게 줄일 수 있다.

---

## 1. 가장 먼저 분류할 것

장애가 발생하면 다음 네 가지부터 정리한다.

```text
1. 누가 누구에게 통신하는가?
2. 이름, Service IP, Pod IP 중 무엇으로 접근하는가?
3. 전체 Pod에서 실패하는가, 특정 Pod·Node에서만 실패하는가?
4. 연결 자체가 실패하는가, 연결 후 응답만 늦거나 끊기는가?
```

범위에 따라 먼저 볼 위치가 달라진다.

| 실패 범위 | 먼저 볼 곳 |
| --- | --- |
| 특정 Pod만 실패 | Application Port, Pod IP·Route, NetworkPolicy |
| 특정 Node의 Pod만 실패 | CNI Agent, Node Route, Tunnel, MTU |
| Service만 실패하고 Pod IP는 정상 | Service, EndpointSlice, kube-proxy 또는 CNI Service 처리 |
| 이름으로만 실패 | Pod의 DNS 설정, CoreDNS, DNS용 Service |
| 외부 통신만 실패 | Egress Policy, Default Route, SNAT, Conntrack |
| 외부 유입만 실패 | LoadBalancer/Gateway, NodePort, Endpoint, Source IP 보존 설정 |

---

## 2. 기본 분석 순서

### 2.1 목적지 Application 확인

먼저 목적지 Pod가 살아 있고 실제 Port를 Listen하는지 확인한다.

```bash
kubectl get pod -o wide
kubectl describe pod <pod>
kubectl exec <pod> -- ss -lnt
```

Application이 Listen하지 않으면 CNI나 Service를 분석해도 해결되지 않는다.

### 2.2 주소별로 직접 호출

같은 요청을 이름, ClusterIP, Pod IP 순서로 테스트하면 실패 구간을 나눌 수 있다.

```text
Service 이름 실패 + ClusterIP 성공 → DNS 구간
ClusterIP 실패 + Pod IP 성공      → Service 구간
Pod IP도 실패                     → Policy 또는 Pod Network 구간
```

### 2.3 단일 지점인지 범위 문제인지 확인

다른 Pod와 다른 Node에서도 같은 요청을 보낸다.

```text
특정 Client Pod만 실패 → Client 설정 또는 Egress Policy
특정 Node에서만 실패   → 해당 Node의 CNI, Route, Tunnel
모든 위치에서 실패     → 공통 Service, DNS, Policy 또는 Server
```

### 2.4 왕복 경로 확인

요청 Packet이 도착해도 Reply Route나 NAT 상태가 잘못되면 연결은 실패한다. 따라서 송신 경로뿐 아니라 응답이 원래 Client로 돌아가는지도 확인한다.

---

## 3. 통신 유형별로 어디를 볼 것인가

### 3.1 Pod → Pod

```text
Source Pod
  → veth
  → Source Node Route / eBPF
  → Node 간 Route 또는 Overlay
  → Destination Node
  → veth
  → Destination Pod
```

확인 순서는 다음과 같다.

1. 목적지 Pod IP와 Port가 맞는지 본다.
2. Source Pod의 Interface와 Route를 본다.
3. Source와 Destination의 NetworkPolicy를 본다.
4. 같은 Node만 되는지, 다른 Node도 되는지 비교한다.
5. 다른 Node에서만 실패하면 Node Route, Tunnel, CNI Agent를 본다.

```bash
kubectl get pod -o wide
kubectl exec <pod> -- ip addr
kubectl exec <pod> -- ip route
kubectl get networkpolicy -A
kubectl get pods -n kube-system -o wide
```

분석 예시:

```text
같은 Node의 Pod끼리는 성공
다른 Node의 Pod끼리는 실패
→ Pod Application과 같은 Node Data Plane은 정상
→ Node 간 Route, Overlay Tunnel, MTU를 우선 확인
```

### 3.2 Pod → Service

```text
Service 이름
  → DNS
  → ClusterIP
  → Service Data Plane
  → EndpointSlice
  → Pod IP
```

확인 순서는 다음과 같다.

1. Service 이름이 ClusterIP로 해석되는지 본다.
2. Service의 `port`와 `targetPort`를 본다.
3. EndpointSlice에 Ready Endpoint가 있는지 본다.
4. Endpoint Pod IP로 직접 호출한다.
5. Pod IP는 되지만 ClusterIP만 안 되면 kube-proxy 또는 CNI의 Service 처리를 본다.

```bash
kubectl get service <service> -o yaml
kubectl get endpointslice -l kubernetes.io/service-name=<service>
kubectl get pods -l <service-selector> -o wide
```

분석 예시:

```text
Pod IP:8080 호출 성공
ClusterIP:80 호출 실패
→ Pod Network와 Application은 정상
→ Service Port, EndpointSlice, Service DNAT 구간을 확인
```

### 3.3 DNS

```text
Application
  → Pod의 /etc/resolv.conf
  → DNS Service
  → CoreDNS
  → Cluster Record 또는 Upstream DNS
```

확인할 곳:

- Pod의 `nameserver`, `search`, `options`
- DNS Service의 ClusterIP와 Endpoint
- CoreDNS Pod 상태와 Log
- UDP/TCP 53을 막는 NetworkPolicy
- 같은 Namespace인지 다른 Namespace인지에 따른 이름 형식

```bash
kubectl exec <pod> -- cat /etc/resolv.conf
kubectl exec <pod> -- nslookup <service>.<namespace>.svc.cluster.local
kubectl get service,endpointslice -n kube-system
kubectl logs -n kube-system -l k8s-app=kube-dns
```

분석 예시:

```text
Service 이름 호출 실패
ClusterIP 직접 호출 성공
→ Service와 Pod Network는 정상
→ resolv.conf, CoreDNS, DNS Policy를 확인
```

### 3.4 Pod → 외부 Network

```text
Pod
  → Egress Policy
  → Node Default Route
  → SNAT 또는 Egress Gateway
  → 외부 Network
  → Conntrack을 통한 Reply
```

확인할 곳:

- 외부 도메인이라면 DNS가 먼저 정상인지
- NetworkPolicy의 Egress 허용 여부
- Pod와 Node의 Default Route
- SNAT 적용 여부
- 방화벽과 Security Group
- Conntrack에 요청과 Reply 상태가 만들어지는지

분석 예시:

```text
외부 IP 호출 성공
외부 도메인 호출 실패
→ Egress Route와 NAT는 정상
→ DNS 구간을 확인
```

### 3.5 외부 Client → Service

```text
External Client
  → LoadBalancer / Gateway
  → NodePort 또는 Service Data Plane
  → Endpoint Pod
  → Reply
```

확인할 곳:

- LoadBalancer 또는 Gateway가 올바른 Service를 가리키는지
- Listener, Port, Health Check 결과
- Service와 EndpointSlice
- `externalTrafficPolicy`와 Local Endpoint 존재 여부
- Client Source IP 보존과 SNAT 여부
- Reply가 같은 경로로 돌아가는지

분석 예시:

```text
Cluster 내부의 Service 호출은 성공
외부에서만 실패
→ Pod Network보다 LoadBalancer, Gateway, NodePort 구간을 먼저 확인

externalTrafficPolicy: Local
특정 Node로 들어올 때만 실패
→ 해당 Node에 Ready Local Endpoint가 있는지 확인
```

---

## 4. 구성요소별 핵심 확인 지점

| 구성요소 | 확인할 핵심 |
| --- | --- |
| Application | Process가 살아 있는가, 올바른 주소와 Port를 Listen하는가 |
| Pod Network | Pod IP, Interface, Route, 같은 Node와 다른 Node의 차이 |
| CNI | Agent 상태, Interface 생성, Node Route, Tunnel 상태 |
| IPAM | Pod IP 할당 여부, IP Pool 여유, 중복 IP, Node Pod CIDR |
| Service | Selector, `port`, `targetPort`, ClusterIP |
| EndpointSlice | Endpoint 존재 여부, 주소·Port, Ready 상태 |
| DNS | `resolv.conf`, DNS Service/Endpoint, CoreDNS 상태와 Log |
| NetworkPolicy | 선택된 Pod, Ingress/Egress 방향, Selector와 Port |
| NAT/Conntrack | SNAT/DNAT 적용, Connection 상태, Reply 역변환 |
| MTU | 작은 Packet은 되고 큰 Packet만 실패하는지, Tunnel Header 크기 |

### NetworkPolicy를 볼 때 주의할 점

통신에는 Source의 Egress와 Destination의 Ingress가 모두 허용되어야 한다.

```text
Source Egress 허용 AND Destination Ingress 허용 → 통신 가능
```

여러 Policy의 허용 규칙은 합집합이다. 또한 같은 항목 안의 `namespaceSelector`와 `podSelector`는 AND이고, 서로 다른 항목으로 쓰면 OR이다.

### MTU를 의심할 때

Overlay는 Tunnel Header를 추가한다. 작은 요청은 되지만 큰 응답이나 특정 TLS 연결만 멈춘다면 Interface MTU, Tunnel MTU, Path MTU Discovery를 확인한다.

```text
작은 Packet 성공 + 큰 Packet 실패
→ 단순 연결 여부보다 MTU와 Fragmentation을 우선 확인
```

---

## 5. 장애 분석 예시

### 예시: Service가 연결되지 않는다

```text
1. EndpointSlice에 Endpoint가 있는가?
   ├─ 없음 → Service Selector, Pod Label, Readiness 확인
   └─ 있음
       ↓
2. Endpoint Pod IP로 직접 연결되는가?
   ├─ 실패 → Application, NetworkPolicy, Pod Network 확인
   └─ 성공
       ↓
3. ClusterIP로 연결되는가?
   └─ 실패 → Service Port와 Service Data Plane 확인
```

### 예시: 특정 Node에서만 통신이 실패한다

```text
1. 실패 Pod가 같은 Node에 몰려 있는지 확인
2. 해당 Node의 CNI Agent 상태 확인
3. Node Route와 Tunnel Interface를 정상 Node와 비교
4. Cross-Node Traffic만 실패하면 Overlay와 MTU 확인
```

### 예시: 연결은 되지만 응답이 돌아오지 않는다

```text
1. 목적지에 요청 Packet이 도착하는지 확인
2. Application이 Reply를 보내는지 확인
3. 목적지의 Reply Route 확인
4. NAT와 Conntrack 상태 확인
5. 비대칭 Routing과 방화벽 Drop 확인
```

---

## 핵심 정리

```text
이름만 실패       → DNS
ClusterIP만 실패  → Service / EndpointSlice / Service Data Plane
Pod IP도 실패     → Application / NetworkPolicy / CNI / Routing
특정 Node만 실패  → Node Route / CNI Agent / Tunnel / MTU
외부로만 실패     → Egress Policy / SNAT / Conntrack
외부에서만 실패   → LoadBalancer / Gateway / NodePort / Reply 경로
```

항상 **Application → 이름 → Virtual IP → Pod IP → Node 간 경로 → Reply** 순서로 좁혀 간다.

## 참고 자료

- [Kubernetes Services, Load Balancing, and Networking](https://kubernetes.io/docs/concepts/services-networking/)
- [Kubernetes Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
- [Kubernetes DNS Debugging](https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/)
- [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [CNI Specification](https://github.com/containernetworking/cni/blob/main/SPEC.md)
