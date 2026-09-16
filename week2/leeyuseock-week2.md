# Kubernetes Pod 생성과 Service 통신 흐름

## 핵심 흐름

### Pod 생성

```text
Pod 생성 요청 → Scheduler → kubelet → CRI → Container Runtime → CNI → IPAM → Pod 네트워크 구성
```

### Service 통신

```text
Client Pod → Service IP → kube-proxy가 구성한 규칙 → Backend Pod IP → CNI가 구성한 Pod Network → Pod
```

쿠버네티스 공식 문서에서도 각 Pod가 고유한 클러스터 IP를 가지며, 서로 다른 노드의 Pod끼리도 직접 통신할 수 있는 것을 Kubernetes 네트워크 모델의 기본 원칙으로 설명합니다.

---

## 구체적인 핵심 흐름

### Pod 생성

```text
Pod 생성 요청
     ↓
Scheduler
Node 선택
     ↓
kubelet
     ↓ CRI
Container Runtime
     ↓
CNI
     ↓
IPAM / Interface / Route 설정
     ↓
Pod IP 생성
     ↓
Pod Network 연결 완료
```

### 실제 Service 통신

```text
Client Pod
     ↓
Service IP
     ↓
kube-proxy가 구성한 규칙
     ↓
Backend Pod IP 선택
     ↓
CNI가 구성한 Pod Network
     ↓
Backend Pod
```

---

## 각 구성 요소의 역할

**CNI**

> Pod IP까지 어떻게 실제로 갈 것인가?

**kube-proxy**

> Service로 온 요청을 어떤 Pod IP로 보낼 것인가?

**Service**

> 변할 수 있는 여러 Pod 앞에 안정적인 접근 주소를 제공한다.

---

## 전체 흐름 정리

> Pod 생성 요청이 들어오면 Scheduler가 실행할 Node를 선택하고, 해당 Node의 kubelet이 CRI를 통해 Container Runtime에 Pod 생성을 요청합니다.
>
> Runtime은 Pod의 네트워크 환경을 준비하고 CNI를 호출하며, CNI는 IPAM을 통해 Pod IP를 할당하고 인터페이스와 라우팅 등을 구성하여 Pod가 통신할 수 있도록 합니다.
>
> 이후 Service를 사용하는 통신에서는 kube-proxy가 구성한 네트워크 규칙을 통해 Service 트래픽의 Backend Pod를 결정하고, 실제 해당 Pod까지의 통신은 CNI가 구성한 Pod Network를 통해 이루어집니다.
