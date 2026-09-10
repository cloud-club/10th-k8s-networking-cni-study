# Week 2. Kubernetes Networking & CNI Fundamentals

## 1. 이번 주 학습 목표

Kubernetes Networking Model을 이해하고, Pod가 생성될 때 네트워크가 어떻게 구성되는지 알아본다.

특히 kubelet, CRI, Container Runtime, CNI, Pod CIDR, IPAM, Service, EndpointSlice, kube-proxy의 역할을 살펴보고, 최종적으로 Service IP로 들어온 요청이 실제 Pod까지 전달되는 흐름을 이해하는 것을 목표로 한다.

---

## 2. Kubernetes Networking Model

Kubernetes Networking은 Cluster에서 발생하는 통신을 크게 다음 4가지 관점으로 볼 수 있다.

### 1. Container ↔ Container

같은 Pod 안의 Container들은 하나의 Network Namespace를 공유한다.

따라서 동일한 Pod IP와 Network Interface를 사용하며, 서로 `localhost`를 통해 통신할 수 있다.

같은 IP와 Port 공간을 공유하므로 각 Application은 서로 다른 Port를 사용한다.

### 2. Pod ↔ Pod

Kubernetes에서는 각 Pod가 고유한 Pod IP를 가진다.

Pod가 같은 Node에 있든 다른 Node에 있든 Cluster Network가 정상적으로 구성되어 있다면 Pod IP를 기반으로 서로 통신할 수 있다.

실제 전달 방식은 사용하는 Network Plugin에 따라 달라질 수 있다.

### 3. Pod ↔ Service

Pod는 생성과 삭제에 따라 IP가 변경될 수 있다.

Service는 여러 Backend Pod에 대해 일정한 접근 주소를 제공하여 Client가 개별 Pod IP를 계속 추적하지 않고 Application에 접근할 수 있도록 한다.

### 4. External ↔ Service

Cluster 외부의 Client가 내부 Application에 접근하기 위한 대표적인 방식은 다음과 같다.

* **NodePort** : Node의 특정 Port를 통해 Service 노출
* **LoadBalancer** : 외부 Load Balancer를 통해 Service 노출
* **Ingress** : HTTP/HTTPS 요청을 Host 또는 Path 기준으로 Service에 Routing

Ingress는 HTTP/HTTPS Routing Rule을 정의하는 Kubernetes Resource이며, 실제 Traffic 처리는 Ingress Controller가 담당한다.

### 정리

| 통신 유형                 | 핵심                                          |
| --------------------- | ------------------------------------------- |
| Container ↔ Container | 같은 Pod의 Container가 Network Namespace와 IP 공유 |
| Pod ↔ Pod             | Pod IP를 기반으로 직접 통신                          |
| Pod ↔ Service         | Service를 통해 Backend Pod 집합에 접근              |
| External ↔ Service    | 외부 Traffic을 Cluster 내부 Application으로 전달     |

Kubernetes는 이러한 통신이 가능해야 한다는 Network Model을 정의하고, 실제 네트워크 구현은 CNI 기반 Network Plugin이 담당한다.

---

## 3. Pod 생성과 Network 구성 흐름

Pod가 생성될 때는 Container와 함께 Network Namespace, Network Interface, IP Address, Routing 정보가 준비된다.

### 주요 구성요소

| 구성요소              | 역할                                    |
| ----------------- | ------------------------------------- |
| kube-apiserver    | Kubernetes API 요청 처리                  |
| Scheduler         | Pod가 실행될 Worker Node 선택               |
| kubelet           | 자신에게 배정된 Pod 실행 및 상태 관리               |
| CRI               | kubelet과 Container Runtime 사이의 표준 API |
| Container Runtime | Pod Sandbox와 Container 관리             |
| CNI               | Pod Network 구성을 위한 표준 Interface       |
| CNI Plugin        | Interface·IP·Route 등 실제 Network 구성    |

### kube-apiserver

`kubectl apply`와 같은 명령은 Kubernetes API를 통해 kube-apiserver로 전달된다.

kube-apiserver는 인증·인가·Admission 등의 과정을 거쳐 Resource 상태를 처리하고, 이후 Scheduler와 kubelet 등의 Component가 해당 상태를 기반으로 동작한다.

```text
kubectl
   ↓
kube-apiserver
   ↓
Scheduler
   ↓
kubelet
```

### kubelet

kubelet은 각 Worker Node에서 동작하는 Agent이다.

자신의 Node에 배치된 Pod를 실행하고 상태를 관리하며, CRI를 통해 Container Runtime에 Pod 실행을 요청한다.

### CRI

CRI(Container Runtime Interface)는 **kubelet과 Container Runtime 사이의 표준 API 규격**이다.

```text
kubelet
   ↓ CRI
containerd / CRI-O
```

containerd와 CRI-O는 CRI를 구현하는 Container Runtime이다.

CRI를 통해 kubelet은 공통된 API 방식으로 여러 Runtime과 통신할 수 있다.

### Container Runtime

Container Runtime은 Pod Sandbox와 Container의 실행 및 상태를 관리한다.

Pod 생성 과정에서는 Pod가 사용할 Network Namespace를 포함한 Sandbox 환경을 준비하고, 이후 CNI Plugin을 호출하여 Network를 구성한다.

### Network Namespace

Network Namespace는 독립적인 Network Stack을 제공하는 격리 공간이다.

하나의 Network Namespace에는 다음과 같은 정보가 존재한다.

* Network Interface
* IP Address
* Routing Table
* ARP / Neighbor Table
* localhost

Pod의 Container들은 같은 Network Namespace를 공유한다.

### CNI

CNI(Container Network Interface)는 Pod Network를 구성하기 위한 표준 Interface이다.

Container Runtime은 CNI 규격에 따라 Network Plugin을 호출하고, CNI Plugin은 다음과 같은 Network 설정을 수행한다.

* Network Interface 구성
* Pod IP 설정
* Routing 설정
* IPAM 연동

```text
kubelet
   ↓ CRI
Container Runtime
   ↓ CNI
CNI Plugin
```

### Pod 생성 흐름

```text
Scheduler가 실행 Node 선택
        ↓
kubelet이 CRI를 통해 Runtime에 Sandbox 생성 요청
        ↓
Runtime이 Pod Sandbox와 Network Namespace 준비
        ↓
Runtime이 CNI Plugin 호출
        ↓
CNI Plugin이 IPAM과 연동
        ↓
Interface·IP·Route 구성
        ↓
Application Container 실행
```

---

## 4. Pod Network / Pod CIDR / IPAM

### Pod Network

Pod Network는 Cluster 내부에서 Pod들이 Pod IP를 기반으로 통신할 수 있도록 구성된 Network이다.

### Pod CIDR

Pod CIDR은 Pod IP로 사용할 수 있는 주소 범위이다.

일반적인 구성에서는 Cluster의 Pod 주소 범위를 Node별로 나누어 사용할 수 있다.

```text
Cluster Pod CIDR
10.244.0.0/16
       ↓
Node A : 10.244.1.0/24
Node B : 10.244.2.0/24
       ↓
개별 Pod IP
```

| Node   | Pod CIDR        | Pod IP 예시     |
| ------ | --------------- | ------------- |
| Node A | `10.244.1.0/24` | `10.244.1.10` |
| Node B | `10.244.2.0/24` | `10.244.2.10` |

CNI에 따라 자체 IP Pool이나 Cloud Network의 IP를 사용하는 방식도 존재한다.

### IPAM

IPAM(IP Address Management)은 사용할 IP를 선택하고 할당·회수하는 기능이다.

```text
Pod 주소 범위
     ↓
IPAM이 IP 할당
     ↓
CNI Plugin이 Interface에 설정
     ↓
Pod IP 사용
```

역할을 정리하면:

* **Pod Network** : Pod 간 통신을 제공하는 전체 Network
* **Pod CIDR** : Pod IP에 사용할 주소 범위
* **IPAM** : 실제 IP Address 할당·회수
* **CNI Plugin** : Interface·IP·Route 등의 Network 설정

---

## 5. Service

Pod가 재생성되면 Pod IP가 변경될 수 있다.

Service는 Backend Pod 집합에 대해 일정한 주소와 Port를 제공한다.

### ClusterIP

ClusterIP는 Cluster 내부에서 Service에 접근하기 위한 Virtual IP이다.

예:

```text
Service
10.96.0.10:80

Backend Pods
10.244.1.10:8080
10.244.2.10:8080
```

Cluster 내부의 Client는 개별 Pod IP 대신 Service의 ClusterIP를 통해 Backend Pod에 접근할 수 있다.

### Label과 Selector

Pod에는 Resource를 구분하기 위한 Label을 설정할 수 있다.

```yaml
metadata:
  labels:
    app: backend
```

여기서:

```text
app      = Label Key
backend  = Label Value
```

Service에서는 Selector를 통해 원하는 Label을 가진 Pod를 선택한다.

```yaml
selector:
  app: backend
```

`app=backend` 조건에 맞는 Pod들이 Service의 Backend 후보가 된다.

### EndpointSlice

EndpointSlice에는 Service의 Backend로 연결되는 Pod들의 IP Address, Port, Ready 상태 등의 Endpoint 정보가 포함된다.

예:

```text
EndpointSlice

10.244.1.10:8080
10.244.2.10:8080
```

Endpoint에는 IP Address, Port, Ready 상태 등의 정보가 포함될 수 있다.

EndpointSlice Controller는 Service의 Selector와 Pod 상태를 기반으로 EndpointSlice를 갱신한다.

```text
Service Selector + Pod Label
            ↓
EndpointSlice Controller
            ↓
EndpointSlice
```

### 관계 정리

```text
Pod Label
    ↓
Service Selector
    ↓
EndpointSlice
    ↓
Backend Endpoint
```

| 요소            | 역할                          |
| ------------- | --------------------------- |
| Label         | Pod 등의 Resource 분류          |
| Selector      | Label을 이용한 Backend 선택 조건    |
| ClusterIP     | Service에 접근하기 위한 Virtual IP |
| EndpointSlice | Backend Endpoint 정보 관리      |

---

## 6. kube-proxy

kube-proxy는 각 Node에서 동작하며 Service Traffic 처리를 위한 Network Rule을 관리한다.

Service와 EndpointSlice의 변경 정보를 확인하고 이를 Node의 Network Rule에 반영한다.

```text
Service / EndpointSlice
        ↓
kube-proxy
        ↓
Node Network Rule
```

iptables 또는 nftables 기반 환경에서는 실제 Packet이 들어왔을 때 Linux Kernel이 해당 Rule을 적용한다.

### DNAT

DNAT(Destination Network Address Translation)는 Packet의 목적지 주소 또는 Port를 변경하는 방식이다.

예:

```text
Before
Dst: 10.96.0.10:80

After
Dst: 10.244.1.10:8080
```

Cilium과 같은 Network Solution은 eBPF를 이용하여 kube-proxy의 Service 처리 기능을 대체할 수도 있다.

---

## 7. Service → Pod Packet Flow

다음 환경을 가정한다.

```text
Service
10.96.0.10:80

Pod A
10.244.1.10:8080

Pod B
10.244.2.10:8080
```

### 1. Client가 Service IP로 요청

```text
Dst: 10.96.0.10:80
```

### 2. Service Network Rule 적용

Node의 Network Stack에서 kube-proxy가 구성한 Service Rule이 적용된다.

### 3. Backend 선택

EndpointSlice에 등록된 Backend를 기반으로 전달 대상이 결정된다.

### 4. Destination 변경

iptables 기반 구현에서는 DNAT를 통해 목적지가 실제 Pod IP와 Port로 변경된다.

```text
10.96.0.10:80
      ↓
10.244.1.10:8080
```

### 5. Pod Network를 통해 전달

목적지가 Pod IP로 결정되면 Pod Network를 통해 해당 Backend Pod까지 전달된다.

```text
Client
   ↓
Service ClusterIP
   ↓
Service Network Rule
   ↓
Backend Pod IP
   ↓
Pod Network
   ↓
Backend Pod
```

### 역할 정리

* **Service** : 안정적인 접근 주소 제공
* **Selector** : Backend Pod 선택 조건
* **EndpointSlice** : Backend Endpoint 정보 관리
* **kube-proxy** : Service Network Rule 구성
* **Linux Kernel** : Rule에 따라 Packet 처리
* **Pod Network** : Backend Pod까지 Packet 전달

---

## 8. Ingress

Ingress는 HTTP/HTTPS 요청을 Host 또는 Path 기준으로 Service에 Routing하기 위한 Kubernetes Resource이다.

예:

```text
example.com/api
        ↓
backend-service

example.com/image
        ↓
image-service
```

Ingress에는 Routing Rule을 정의한다.

```yaml
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /api
            backend:
              service:
                name: backend-service
                port:
                  number: 80
```

Ingress Controller는 이 Rule을 확인하여 실제 Traffic 처리 설정에 반영한다.

```text
External Client
      ↓
Ingress Controller
      ↓
Service
      ↓
Backend Pod
```

---

## 9. 전체 흐름

### Pod Network 준비

```text
Scheduler
   ↓
kubelet
   ↓ CRI
Container Runtime
   ↓
Network Namespace
   ↓ CNI
CNI Plugin + IPAM
   ↓
Pod Network 구성
```

### Service 준비

```text
Pod Label / IP / Ready 상태
        ↓
EndpointSlice Controller
        ↓
EndpointSlice
        ↓
kube-proxy
        ↓
Service Network Rule
```

### 실제 통신

```text
Client
   ↓
Service ClusterIP
   ↓
Service Network Rule
   ↓
Backend Pod IP
   ↓
Pod Network
   ↓
Backend Pod
```


---

## 10. 참고 자료

* Kubernetes Networking 공식 문서
  https://kubernetes.io/ko/docs/concepts/cluster-administration/networking/

* Kubernetes Container Runtime Interface
  https://kubernetes.io/docs/concepts/containers/cri/

* Kubernetes Service
  https://kubernetes.io/docs/concepts/services-networking/service/

* Kubernetes 네트워크 정리
  https://yozm.wishket.com/magazine/detail/2251/
