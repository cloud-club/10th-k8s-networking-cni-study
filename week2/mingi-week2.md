# Kubernetes Networking & CNI Fundamentals - 2 week

## Kubernetes Networking Model

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/0f078362-5a59-4c6b-929c-d1855ba7748b" />


### 통신 유형 4가지

1. **같은 Pod 안 컨테이너 ↔ 컨테이너 — `localhost`**
2. **Pod ↔ Pod — 모델의 핵심, CNI가 담당**
3. **Pod ↔ Service — kube-proxy가 담당**
4. **외부 ↔ Service — NodePort / LoadBalancer / Ingress**

### 1. 같은 Pod 안 컨테이너 ↔ 컨테이너

- IP는 `localhost` , `Port`로 컨테이너 구분

<p align="center"><img width="400" alt="image" src="https://github.com/user-attachments/assets/6525675f-707e-4041-b652-8acb4c39d934" />


### 2. Pod ↔ Pod

- br0(Bridge)가 L2 스위치 역할

<p align="center"><img width="700" alt="image" src="https://github.com/user-attachments/assets/bd186e09-6e7b-45eb-bd27-87f9a0b93604" />


### 3. Pod ↔ Service

#### **Service (ClusterIP)**

- **파드 집합에서 실행중인 애플리케이션을 네트워크 서비스로 노출하는 추상화 방법**
- Service 타입
    - ClusterIP: 서비스를 클러스터 내부 IP에 노출하는 Service 타입
    - NodePort: 각 노드의 IP에서 고정된 포트(`NodePort`)로 서비스를 노출하는 타입
    - LoadBalancer: 외부 로드 밸런서를 사용하여 서비스를 외부에 노출하는 타입
    - ExternalName: 외부로 나가는 트래픽을 변환(도메인 이름을 변환)하는 용도

#### ClusterIP

- **Pod IP는 배포·스케일링·재스케줄 때마다 바뀌는데 이것을 고정된 엔드포인트로 만드는 역할**
- 역할
    - 변하지 않는 주소 제공
    - 여러 Pod로 트래픽 분배
    - Ready 상태인 Pod만 대상에 포함
- 동작 방식
    - Service에는 `Selecter`만 있고, 실제 Pod IP 목록은 `EndpointSlice`에 컨트롤러가 계속 갱신
    - 각 노드의 kube-proxy가 `EndpointSlice` 를 watch해서 커널에 DNAT 규칙 씀
    - 패킷이 ClusterIP로 오면 커널이 목적지를 실제 Pod IP로 바꿈
- `clusterip-example.ymal`
    
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: my-svc
      namespace: default
    spec:
      type: ClusterIP          # 생략해도 기본값
      clusterIP: 10.96.0.100   # 보통은 비워두고 API 서버가 자동 할당한 값이 채워짐
      selector:
        app: nginx             # 이 라벨을 가진 Pod들이 대상
      ports:
      - name: http
        protocol: TCP
        port: 80               # ClusterIP에서 열리는 포트
        targetPort: 8080       # Pod가 실제로 리스닝하는 포트 
    ```
    

### **kube-proxy**

- **노드에서 네트워크 규칙을 유지, 관리**
- 역할
    - API 서버를 watch → Service와 EndpointSlice 변화를 구독
    - 그 결과를 자기 노드의 커널에 규칙(iptables)으로 기록
- 모드
    
    
    | userspace | kube-proxy가 직접 트래픽을 중계하던 초기 방식. 느리고 단일 장애점이라 1.26에서 제거됨 |
    | --- | --- |
    | iptables | 커널 nat 테이블에 DNAT 규칙을 써두는 현재 기본값.  |
    | IPVS | 커널 내장 L4 로드밸런서를 사용. 해시 기반이라 대규모에서 빠르고 분배 알고리즘도 선택 가능 |
    | nftables | iptables의 후속 프레임워크로 같은 일을 수행. 대규모 환경의 성능 한계를 개선한 신규 모드 |

#### Pod ↔ Service 내부 통신

<p align="center"><img width="600" alt="image" src="https://github.com/user-attachments/assets/5aff200a-810f-4f3f-97b1-661bad6669df" />


```bash
1. Pod1이 ClusterIP(10.96.0.100:80)로 패킷을 내보냄
   - 목적지가 자기 서브넷(10.0.1.0/24) 밖이므로 default 경로 선택
   - 목적지 MAC = 게이트웨이인 br0의 MAC
   - Pod는 이게 Service인지 외부 주소인지 모름

2. br0 도착 → 목적지 MAC이 자기 것이므로 "로컬 수신" 판정
   - 이더넷 헤더 제거 = L2 → L3 계층 상승
   - 남은 IP 패킷을 호스트 커널의 IP 스택으로 전달

3. netfilter/PREROUTING 진입 (라우팅 이전)
   - kube-proxy가 미리 써둔 규칙에 ClusterIP가 매칭
   - DNAT 실행: 목적지 10.96.0.100 → 10.0.1.3
   - 출발지 10.0.1.2는 그대로 (Pod→Pod이므로 MASQUERADE 제외)

4. conntrack에 변환 내역 기록
   - 응답 패킷의 역변환에 사용됨

5. 라우팅 조회 — 이 시점의 목적지는 이미 10.0.1.3
   - "10.0.1.0/24 dev br0" 매칭 → br0로 내보내기로 결정

6. 이더넷 헤더 신규 생성 = L3 → L2 계층 하강
   - 출발지 MAC = br0, 목적지 MAC = Pod2 (ARP로 획득)
   - IP 헤더는 계속 하나가 유지되고 목적지 필드만 바뀐 상태

7. br0가 L2 스위치로서 veth1 포트로 전달 → Pod2 도착
   - Pod2가 보는 출발지는 10.0.1.2 (클라이언트 Pod IP 보존)

--- 응답 ---

8. Pod2가 10.0.1.3 → 10.0.1.2 Pod1로 응답 송신

9. 커널이 conntrack 항목을 찾아 출발지를 역변환
   - 10.0.1.3 → 10.96.0.100
   - 응답 방향에는 별도 iptables 규칙이 없음. 
   - 규칙은 첫 패킷에만 평가되고 이후는 conntrack 항목을 따라 자동 처리

10. Pod1 도착 — 자기가 요청한 10.96.0.100에서 응답이 온 것으로 인식
```

---

### 4. 외부 ↔ Service

#### NodePort

- **모든 노드의 특정 포트를 열어 외부 접근을 허용하는 Service 타입**
- 동작
    - Service를 만들면 기본 범위인 30000–32767 범위에서 포트 하나가 할당되고, 클러스터의 모든 노드가 그 포트를 연다
    - 할당된 포트로 트래픽이 들어오면 매핑된 ClusterIP로 전달 (NAT)
- 클라이언트가 노드 IP를 알아야한다, LB 뒷단으로 대부분 존재

#### LoadBalancer

- **클라우드 프로바이더에게 외부 로드밸런서를 요청하는 Service 타입**
- 동작
    - Service를 만들면 cloud-controller-manager가 감지해서  외부 LB를 프로비저닝
    - 각 노드의 NodePort를 그 LB의 백엔드로 등록
    - 클라이언트 → 클라우드 LB → 노드들의 NodePort → DNAT → Pod
- Service 하나 당 LB 한계 (마이크로서비스 20개 → LB 20개)

#### Ingress

- **HTTP 호스트명·경로 기준으로 트래픽을 여러 Service에 나눠주는 L7 라우팅 규칙**
- Ingress : 규칙 선언
- Ingress Controller : 실제 처리, Ingress 리소스를 watch하고 자기 설정을 갱신

```yaml
api.example.com     → svc-api
shop.example.com    → svc-shop
example.com/admin   → svc-admin
```

- 동작
    
    ```yaml
    클라이언트
       ↓
    클라우드 LB  ← LoadBalancer 타입 Service 하나 (컨트롤러 노출용)
       ↓
    Ingress Controller Pod  ← 여기서 Host/Path 판단
       ↓
    ClusterIP Service들
       ↓
    백엔드 Pod들
    ```
    
- LB가 하나만 필요, HTTP(S) 전용

#### 외부 ↔ Service 통신

<p align="center"><img width="800" alt="image" src="https://github.com/user-attachments/assets/d6f5adbc-37c2-440a-a736-5b530998027f" />


```yaml
 1. 클라이언트 → LB → 노드2의 eth0:30080 (10.100.0.3:30080)
   - src=클라이언트IP, dst=10.100.0.3:30080

2. eth0 도착 → 이더넷 헤더 제거 → 호스트 커널 IP 스택 진입

3. nat/PREROUTING (netfilter)
   - KUBE-NODEPORTS에서 포트 30080 매칭
   - DNAT: dst 10.100.0.3:30080 → 10.0.2.2:8080

4. conntrack 기록

5. 라우팅 조회 — 목적지는 이제 10.0.2.2
   - "10.0.2.0/24 dev br0" 매칭 → br0로 결정

6. nat/POSTROUTING (netfilter)
   - MASQUERADE: src 클라이언트IP → 10.0.2.1 (br0 IP)
   - 같은 노드여도 SNAT는 걸림 (Cluster 정책이므로)

7. 이더넷 헤더 생성 → br0 → veth0 → Pod1 도착
   - Pod가 보는 출발지는 노드 주소. 클라이언트 IP는 소실

--- 응답 ---

8. Pod → br0 → 커널 IP 스택

9. conntrack이 두 변환을 역으로 되돌림
   - src 10.0.2.2 → 10.100.0.3:30080
   - dst → 클라이언트IP

10. eth0 → LB → 클라이언트 도착
```

---

## Pod Network

## kubelet

각 워커 노드에서 실행되며 파드(Pod)와 컨테이너가 정상적으로 작동하도록 관리하는 에이전트

- apiserver를 watch 하면서 내 노드에 배정된 파드를 감지
- PodSpec대로 컨테이너가 잘 돌고있는지 Check

## Container Runtime

컨테이너 실행을 담당하는 소프트웨어

- kubelet과는 CRI(Container Runtime Interface)라는 표준 인터페이스로 대화
- CNI 플러그인을 호출하는 주체 (`/etc/cni/net.d` → `/opt/cni/bin`)

## CRI(Container Runtime Interface)

 Kubelet이 다양한 컨테이너 런타임을 사용할 수 있도록 하는 플러그인 인터페이스

- CRI API
    - **`RuntimeService`**: Pod sandbox와 컨테이너의 생성, 실행, 종료, 상태 조회 등을 담당
    - **`ImageService`**: 이미지 다운로드, 목록 조회, 삭제 등을 담당

```yaml
kubelet
   │ CRI (gRPC)
containerd   ← 고수준 런타임 - 이미지 관리, 스토리지, CRI 응답, 생명주기 관리 같은 전반을 담당
   │ OCI Runtime Spec
  runc       ← 저수준 런타임 - 실제로 네임스페이스와 cgroup을 만들어 프로세스를 격리
   │
리눅스 커널 (namespace, cgroup)
```

## OCI(Open Container Initiative)

컨테이너를 실행하는 방법(Runtime)과 컨테이너 이미지를 구성하는 형식(Image Format)에 대한 개방형 표준(Open Standard)을 정의

- 리눅스 재단(Linux Foundation) 산하의 프로젝트
- 다른 컨테이너 도구들이 이미지를 호환하여 만들고, 배포하고, 실행할 수 있도록 공통 표준을 제공하기 위해 사용

<p align="center"><img width="1200" height="559" alt="image" src="https://github.com/user-attachments/assets/16b740a0-a6ac-4c26-8965-0a6af9392f76" />


### IPAM(IP Address Management)

Pod에게 고유한 IP를 할당하고, 중복 없이 관리하는 시스템

- 역할
    - 사용 가능한 IP 탐색
    - Pod에 IP 할당
    - 중복 할당 방지
    - 할당 상태 저장
    - Pod 삭제 시 IP 회수
    - 필요하면 게이트웨이와 라우트 정보 반환
- CIDR 범위

| CNI | 흔히 사용하는 Pod CIDR |
| --- | --- |
| Flannel | `10.244.0.0/16` |
| Calico | `192.168.0.0/16` |
| Cilium | `10.0.0.0/8` |
| Cilium 노드별 CIDR | `/24` |
- IPAM 방식

| IPAM 방식 | 주소를 가져오는 곳 | 특징 |
| --- | --- | --- |
| `host-local` | 노드별 로컬 CIDR | 단순하고 빠르지만 노드별 CIDR 분리가 필요 |
| `static` | CNI 설정에 지정된 IP | 고정 주소, 일반 Pod 대규모 관리에는 부적합 |
| `dhcp` | 외부 DHCP 서버 | 기존 네트워크의 DHCP 이용 |
|  | Calico IPAM | Calico IPPool |
| Cilium IPAM | Cilium Pool 또는 Node CIDR | 다양한 IPAM 모드 지원 |
| Cloud IPAM | AWS VPC, Azure VNet 등 | 클라우드 네트워크 IP를 Pod에 직접 할당 |

#### Kubernetes가 노드별 Pod CIDR을 할당하는 방식

1. `kube-controller-manager`가 노드에 Pod CIDR 블록 할당 (ex. `Node1` → `10.0.1.0/24`)
2. `CNI IPAM` 이 개별 Pod에 IP 할당 (ex. `Pod1` → `10.0.1.2`)

```bash
Cluster Pod CIDR
10.0.0.0/16
      │
      │ kube-controller-manager
      │ Node IPAM Controller
      ▼
Node 1: 10.0.1.0/24
Node 2: 10.0.2.0/24
      │
      │ CNI IPAM
      ▼
Pod IP (Pod 1: 10.0.1.2)
```

#### CNI가 자체 IP Pool을 관리하는 방식

- 역할

```bash
kube-controller-manager
  └── Node CIDR 할당 비활성화 또는 미사용

Calico/Cilium 등의 IPAM
  ├── 전체 IP Pool 관리
  ├── 노드에 IP 블록 분배
  ├── Pod IP 할당
  └── Pod 삭제 시 IP 회수
```

- 동작

```bash
CNI IP Pool
10.0.0.0/16
      │
      │ CNI의 자체 IPAM
      ▼
Node 1용 IP Block
10.0.1.0/26

Node 2용 IP Block
10.0.2.0/26
      │
      ▼
개별 Pod IP 할당
```

### Pod 생성 흐름

<img width="900" alt="image" src="https://github.com/user-attachments/assets/2f8ca440-2ce2-4182-9eb8-0280556227e1" />


- 노션 링크

    https://fascinated-jobaria-aa1.notion.site/Kubernetes-Networking-CNI-Fundamentals-2-week-3d4833a88b7e80b984fec8f82122b482?source=copy_link

- 참고자료
    
    https://kubernetes.io/ko/docs/concepts/cluster-administration/networking/
    
    https://kubernetes.io/ko/docs/concepts/services-networking/
    
    https://medium.com/finda-tech/kubernetes-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EC%A0%95%EB%A6%AC-fccd4fd0ae6
    
    https://bcho.tistory.com/1353
    
    https://velog.io/@sangmiiiiin/IPAMIP-Address-Management-CNI
