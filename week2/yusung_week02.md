## 1. Kubernetes Networking Model

### Kubernetes가 요구하는 통신 방식

- 일반 Pod는 클러스터 내에서 고유한 Pod IP 보유
- 같은 Pod의 컨테이너는 네트워크 네임스페이스 공유
- 같은 노드와 다른 노드의 Pod에 직접 통신 가능
- 직접적인 Pod 간 통신에 NAT 불필요
- 노드의 kubelet 같은 에이전트는 해당 노드의 Pod에 접근 가능
- NetworkPolicy 등에 의한 의도적인 통신 제한은 별도 고려
- Dual-stack 환경에서는 IPv4와 IPv6 주소를 함께 보유 가능

### Pod와 컨테이너의 관계

- 일반적으로 **Pod 하나에 애플리케이션 컨테이너 하나** 배치
- 밀접하게 연동되는 보조 기능이 필요한 경우 sidecar 컨테이너 함께 배치
- 같은 Pod에 여러 컨테이너가 존재하면 IP·인터페이스·포트 공간 공유
- 같은 Pod의 컨테이너끼리는 `localhost`로 통신 가능
- 서로 다른 Pod는 각자의 네트워크 공간 사용
- `hostNetwork: true` Pod는 노드 네트워크를 사용하는 예외

### 네트워크 모델과 구현의 차이

| 구분 | 의미 |
| --- | --- |
| Kubernetes 네트워크 모델 | 어떤 통신이 가능해야 하는지 정의 |
| 네트워크 구현 | 인터페이스·라우팅·터널 등을 통해 실제 통신 구현 |
- Kubernetes 네트워크 모델 자체는 VXLAN이나 특정 CNI 제품을 강제하지 않음
- 네트워크 모델의 요구사항을 충족하는 여러 구현 방식 존재

---

## 2. kubelet / CRI / Runtime / CNI 역할

| 구성요소 | 정체 | 주요 역할 |
| --- | --- | --- |
| kubelet | 노드에서 실행되는 에이전트 | 자신에게 배정된 Pod의 실행과 상태 관리 |
| CRI | kubelet과 런타임 사이의 API 규약 | Sandbox·컨테이너 생성, 시작, 상태 조회 요청 |
| containerd / CRI-O | CRI를 구현하는 컨테이너 런타임 | Pod 실행 환경과 컨테이너 관리, CNI 연동 |
| CNI | 네트워크 플러그인 호출 규약 | 네트워크 연결·해제에 필요한 입력과 결과 정의 |
| CNI 플러그인 | CNI 규약을 구현한 프로그램 | Pod 네트워크 인터페이스와 관련 설정 구성 |
| IPAM | IP 주소 관리 기능 또는 구성요소 | IP 할당·회수·사용 상태 관리 |

```
kubelet
   │
   │ CRI API / gRPC
   ▼
컨테이너 런타임의 CRI 구현
   │
   │ CNI 규약에 따라 플러그인 호출
   ▼
CNI 플러그인
   ├─ IPAM 연동
   ├─ 네트워크 인터페이스 구성
   └─ IP·라우트 등 설정
```

- **CRI:** kubelet이 런타임에 요청하는 방식
- **CNI:** 런타임이 네트워크 플러그인에 요청하는 방식
- CRI와 CNI는 서로 다른 경계의 인터페이스
- CRI 자체가 독립 프로세스로 CNI를 호출하는 구조가 아님
- **CRI 요청을 처리하는 런타임 구현이 CNI 호출**

---

## 3. Pod 생성과 네트워크 준비 과정

### Sandbox의 의미

- **Pod의 컨테이너들이 실행될 기반 환경**
- 네트워크 네임스페이스 등 Pod 단위의 실행 환경 포함
- 네트워크 관점에서는 **컨테이너들이 함께 사용할 네트워크 공간을 준비하는 단계**로 이해
- 일반적인 Linux 런타임에서는 pause 컨테이너를 활용하는 구현 존재
- pause 프로세스 자체가 Pod IP를 할당하는 것은 아님

### Pod 생성 순서

1. **스케줄러가 Pod의 실행 노드 선택**
    - Pod를 실행할 노드 결정
    - 해당 노드의 kubelet이 Pod 실행 진행
2. **kubelet이 런타임에 `RunPodSandbox` 요청**
    - CRI API를 통해 Pod 실행 환경 준비 요청
3. **런타임이 Sandbox와 네트워크 공간 준비**
    - 일반 Pod를 위한 network namespace 준비
    - 이후 CNI가 설정할 대상 네트워크 공간 확보
4. **런타임이 CNI `ADD` 호출**
    - 네트워크 설정과 대상 namespace 등의 정보 전달
5. **CNI 플러그인이 IPAM을 통해 IP 확보**
    - 사용할 주소 풀에서 Pod IP 할당
    - IPAM 방식에 따라 별도 플러그인·노드 에이전트 등과 연동
6. **CNI 플러그인이 Pod 네트워크 구성**
    - 인터페이스 생성 또는 설정
    - IP 주소와 라우트 등 적용
    - 런타임에 CNI 결과 반환
7. **kubelet이 Sandbox 상태 확인**
    - 런타임을 통해 Pod 네트워크 상태와 IP 확인
8. **kubelet이 컨테이너 생성·시작 요청**
    - 준비된 Pod 네트워크를 사용하는 컨테이너 실행
    - Init container가 있는 경우 해당 실행 규칙 적용
    - Pod 상태를 API Server에 반영

### Pod 생성 시퀀스

첨부한 그림과 동일한 핵심 흐름을 표현한 시퀀스.

<img width="772" height="601" alt="image" src="https://github.com/user-attachments/assets/98e64a34-af17-46d6-9784-e3b8b96142e5" />


- 인터페이스 생성과 IP 확보 등의 세부 순서는 플러그인에 따라 차이
- Sandbox 생성과 컨테이너 실행의 내부 절차는 런타임에 따라 차이

---

## 4. Pod Network / Pod CIDR / IPAM

### 주소와 네트워크 용어 구분

| 구분 | 의미 |
| --- | --- |
| Node IP | 노드의 네트워크 주소 |
| Pod IP | Pod의 네트워크 주소 |
| Service ClusterIP | Service 접근을 위한 가상 주소 |
| Pod Network | Pod 간 통신을 제공하는 전체 네트워크 |
| Pod CIDR | Pod 주소에 사용하는 CIDR 범위 |
| `Node.spec.podCIDR` / `spec.podCIDRs` | 노드에 배정된 Pod 주소 범위를 표현하는 필드 |
| Service CIDR | Service ClusterIP 할당에 사용하는 주소 범위 |
| IPAM | 주소 풀에서 실제 IP를 할당·관리하는 기능 |

### Pod IP 할당 관계

```
Pod 주소 범위 또는 IP Pool
            ↓
      IPAM이 IP 할당
            ↓
 CNI 플러그인이 Pod에 설정
            ↓
       Pod IP 사용
```

- Pod CIDR은 **주소 범위**
- IPAM은 **주소 할당과 관리 기능**
- CNI 플러그인은 **할당 결과를 네트워크에 적용하는 구성요소**
- Pod Network는 **Pod들이 연결되어 통신하는 전체 네트워크**

### 노드별 Pod CIDR을 사용하는 경우

```
클러스터의 Pod 주소 범위
            ↓
       노드별 범위 배정
            ↓
  해당 범위에서 개별 Pod IP 할당
```

- 노드에 CIDR을 배정하는 작업과 개별 Pod IP를 할당하는 작업은 별개
- 모든 CNI가 `Node.spec.podCIDR`을 사용하는 것은 아님
- CNI에 따라 자체 IPPool이나 클라우드 네트워크 주소 사용
- Pod IP 할당과 Service ClusterIP 할당은 별도 체계
- IP 할당 성공만으로 다른 노드까지의 통신 성공을 보장하지 않음

---

## 5. CNI의 네트워크 구성 책임 범위

### CNI 플러그인의 주요 작업

- 대상 네트워크 인터페이스 생성 또는 설정
- IPAM 연동
- IP 주소와 라우트 등 적용
- 설정 결과를 런타임에 반환
- 네트워크 해제 시 관련 설정과 IP 할당 정리

### CNI 플러그인과 네트워크 솔루션 전체의 차이

- Calico·Cilium 같은 솔루션에는 플러그인 외에 에이전트·컨트롤러 등 포함
- 구성요소들이 노드 간 경로·정책·터널·eBPF 상태 등을 관리
- CNI 실행 파일은 일반적으로 연결 설정 후 종료
- 이후 패킷은 구성된 네트워크 경로와 규칙을 통해 전달
- **패킷마다 CNI 실행 파일을 호출하는 방식이 아님**

### 기능별 책임 구분

| 기능 | 기본 책임 |
| --- | --- |
| Pod 실행 노드 결정 | 스케줄러 |
| Pod 실행 조정 | kubelet |
| Sandbox와 namespace 준비 | 컨테이너 런타임 |
| Pod 네트워크 연결 | CNI 플러그인과 네트워크 구현 |
| Pod IP 할당·회수 | IPAM |
| Service ClusterIP 할당 | Kubernetes 제어 영역 |
| Service 백엔드 목록 관리 | EndpointSlice 컨트롤러 |
| Service 트래픽 처리 | kube-proxy 또는 대체 구현 |
| Service 이름 해석 | 클러스터 DNS |
| NetworkPolicy 집행 | 지원하는 네트워크 구현 |
- NetworkPolicy는 지원하는 네트워크 구현이 있어야 실제 집행
- Service 처리까지 제공하는 네트워크 솔루션도 존재
- Service 로드밸런싱은 CNI 규약 자체의 필수 기능이 아님

---

## 6. Pod 네트워크를 통한 전달

### 같은 노드의 Pod 간 통신

```
출발지 Pod
    ↓
노드 내부 네트워크 경로
    ↓
목적지 Pod
```

- bridge·라우팅·eBPF 등 구현에 따라 전달 방식 차이
- 일반적으로 노드 간 네트워크를 통과할 필요 없음

### 다른 노드의 Pod 간 통신

```
출발지 Pod
    ↓
출발지 Node
    ↓
노드 간 네트워크
    ↓
목적지 Node
    ↓
목적지 Pod
```

| 전달 방식 | 동작 |
| --- | --- |
| Overlay | Pod 패킷에 터널 헤더를 추가해 노드 간 전달 |
| Native / Routed | 기반 네트워크가 Pod IP로 전달할 수 있도록 경로 구성 |
- Overlay의 캡슐화와 NAT는 다른 개념
- 캡슐화는 원래 패킷을 운반할 바깥쪽 헤더 추가
- Pod 간 NAT 없는 통신과 Overlay 사용은 함께 성립 가능

---

## 7. Service와 백엔드 Pod 연결

### Service가 필요한 이유

- Pod 삭제·재생성에 따라 IP 변경 가능
- 클라이언트가 개별 Pod 주소를 계속 추적하는 부담 발생
- Service를 통해 백엔드 집합에 접근하는 안정적인 주소와 포트 제공

### Service 연결에 사용하는 정보

| 항목 | 역할 |
| --- | --- |
| selector | 백엔드 Pod를 선택하는 라벨 조건 |
| ClusterIP | 클라이언트가 접근하는 Service 가상 IP |
| port | Service가 제공하는 포트 |
| targetPort | 백엔드가 요청을 받는 포트 |
| EndpointSlice | 백엔드 IP·포트·준비 상태 등을 담는 API 객체 |

### 백엔드 연결 정보 준비 순서

1. **Service에 Pod 선택 조건 지정**
    - `selector`에 선택 조건 설정
2. **EndpointSlice 컨트롤러가 조건에 맞는 Pod 확인**
    - 같은 namespace의 Pod 라벨과 selector 비교
3. **Pod 연결 정보를 EndpointSlice에 반영**
    - Pod IP·포트·준비 상태 등 기록
    - Pod 추가·삭제·상태 변경 시 갱신
4. **kube-proxy가 Service와 EndpointSlice 확인**
    - Service 주소와 백엔드 연결 정보 확인
5. **각 노드에 Service 전달 규칙 설정**
    - Service 주소로 들어오는 트래픽을 백엔드로 전달하도록 구성

```
Service의 selector
        +
Pod의 라벨·IP·준비 상태
        ↓
EndpointSlice 컨트롤러
        ↓
EndpointSlice 갱신
        ↓
kube-proxy
        ↓
노드의 전달 규칙 설정
```

- EndpointSlice는 패킷을 전달하는 서버가 아닌 **백엔드 정보 객체**
- EndpointSlice에 주소가 존재해도 트래픽 대상이 된다고 단정 불가
- 준비 상태와 트래픽 정책 등을 고려해 실제 대상 결정
- 기본 흐름은 selector가 있는 일반 ClusterIP Service 기준

---

## 8. kube-proxy의 역할

### 규칙 설정과 패킷 처리 구분

- Service와 EndpointSlice 변경 감시
- 변경 사항을 노드의 네트워크 규칙에 반영
- Linux의 iptables·nftables 모드에서는 커널이 실제 패킷 처리
- kube-proxy 프로세스가 애플리케이션 패킷을 하나씩 받아 전달하는 구조가 아님

```
[설정 흐름]

API Server
    ↓ Service·EndpointSlice 변경
kube-proxy
    ↓
커널 규칙 갱신

[패킷 흐름]

클라이언트 Pod
    ↓
커널이 규칙 적용
    ↓
백엔드 Pod
```

### CNI와 kube-proxy의 연결 관계

| 구분 | 핵심 역할 |
| --- | --- |
| CNI와 Pod 네트워크 구현 | Pod가 네트워크에 연결되고 다른 Pod에 도달할 수 있도록 구성 |
| kube-proxy | Service로 들어온 트래픽을 백엔드 Pod로 전달하도록 규칙 구성 |
- Cilium 등은 설정에 따라 kube-proxy를 대체하는 Service 처리 기능 제공
- 이 경우에도 **Pod 연결 기능**과 **Service 처리 기능**을 개념적으로 구분 가능

---

## 9. Service → Pod Packet Flow

### 요청 전달 순서

1. **클라이언트가 Service 주소 확인**
    - 이름으로 접근하는 경우 클러스터 DNS를 통해 ClusterIP 확인
    - IP로 직접 접근하는 경우 DNS 조회 생략
2. **클라이언트가 Service IP와 포트로 패킷 전송**
    - 개별 백엔드 Pod IP를 알 필요 없음
3. **노드의 Service 처리 규칙 적용**
    - 준비 상태와 정책에 맞는 백엔드 중 대상 선택
4. **목적지 주소와 포트를 백엔드 기준으로 변환**
    - Service IP → Pod IP
    - Service port → 백엔드 포트
    - 이 목적지 변환을 DNAT로 표현
5. **변환된 목적지에 따라 Pod 네트워크로 전달**
    - 같은 노드의 Pod라면 노드 내부 경로 사용
    - 다른 노드의 Pod라면 노드 간 경로 사용
6. **백엔드 애플리케이션이 요청 수신**

```
클라이언트
    │ 목적지: Service IP:Port
    ▼
노드의 Service 처리 규칙
    │ 백엔드 선택 및 DNAT
    │ 목적지: Pod IP:Backend Port
    ▼
Pod 네트워크
    ▼
백엔드 Pod
    ▼
애플리케이션
```

### 응답 전달 순서

1. **백엔드 Pod가 응답 전송**
2. **응답 경로에서 기존 연결의 NAT 매핑 적용**
3. **Service 주소와 통신하는 형태로 클라이언트에 응답 전달**
    - 일반적인 netfilter NAT 경로에서는 conntrack이 연결과 변환 상태 추적
    - 같은 TCP 연결의 후속 패킷에는 기존 매핑 적용
    - TCP 패킷마다 백엔드를 새로 선택하는 방식이 아님
    - HTTP keep-alive로 연결을 재사용하면 여러 요청이 같은 백엔드로 전달 가능
    - 추가 SNAT·외부 유입·eBPF 기반 처리에서는 주소와 경로 차이 존재

---

## 10. 전체 흐름

전체 과정을 **Pod 네트워크 준비 → Service 전달 규칙 준비 → 실제 패킷 전달**로 구분.

```
[1. Pod 네트워크 준비]

스케줄러가 실행 노드 선택
        ↓
kubelet이 CRI를 통해 런타임에 Sandbox 생성 요청
        ↓
런타임이 Pod 네트워크 공간 준비
        ↓
런타임이 CNI 플러그인 호출
        ↓
CNI 플러그인이 IPAM과 연동
        ↓
Pod IP·인터페이스·라우트 설정
        ↓
애플리케이션 컨테이너 실행

[2. Service 전달 규칙 준비]

Pod IP·라벨·준비 상태
        ↓
EndpointSlice 컨트롤러가 백엔드 정보 반영
        ↓
kube-proxy가 Service와 EndpointSlice 확인
        ↓
노드의 Service 전달 규칙 설정

[3. 실제 패킷 전달]

클라이언트가 Service 주소로 요청
        ↓
노드의 규칙에 따라 백엔드 Pod 선택
        ↓
목적지를 백엔드 Pod IP와 포트로 변환
        ↓
Pod 네트워크를 통해 백엔드 Pod까지 전달
```

- 앞의 두 과정은 **통신을 위한 준비 작업**
- 마지막 과정은 **준비된 규칙과 경로를 사용하는 실제 통신**
- 패킷마다 kubelet → CRI → CNI를 다시 호출하는 구조가 아님
- 실제 시스템에서는 각 구성요소가 변경 사항을 비동기적으로 반영
- Service가 Pod보다 먼저 생성되는 경우도 가능

### 전체 흐름에서 구분할 핵심 상태

```
Pod 실행 노드 결정
        ↓
Sandbox 준비
        ↓
Pod IP 및 네트워크 설정
        ↓
애플리케이션 실행
        ↓
트래픽을 받을 준비 완료
        ↓
EndpointSlice 상태 반영
        ↓
Service 전달 규칙 반영
```

- Pod IP 할당 완료와 애플리케이션 준비 완료는 별개
- 컨테이너 실행과 readiness 통과는 별개
- EndpointSlice 갱신과 모든 노드의 규칙 반영은 별개
- 각 구성요소의 상태가 비동기적으로 연결되는 구조
