# W6. Multi-Networking &amp; Secondary Networking

## 1. Multus/멀티 네트워크는 왜 필요할까?

같은 Pod가 서로 다른 용도의 네트워크에 동시에 연결되어야 한다는 가정은 어떤 상황에서 생기는지 고민해봤습니다. OpenShift Data Foundation의 Multus 용례를 보면 스토리지 통신을 전용 인터페이스와 네트워크에 분리하는 구성이 있습니다. 스토리지의 일반 통신은 기본 클러스터망을 사용하고, 대용량 데이터 복제는 전용 스토리지망을 별개로 연결하여 대규모 트래픽을 처리할 수도 있을 것입니다. Pod가 고유의 클러스터 통신이 있음에도 내부망/폐쇄망에 붙어야 해서 멀티 네트워킹이 필요할 수도 있고, 관리/설정을 전담하는 통신과 일반 사용자 패킷을 처리하는 통신을 분리하는 구성도 가능할 것 같습니다. 다만 인터페이스를 늘렸다고 해서 대역폭 자체가 늘어나는 건 아니라서 목적에 따른 NIC와 하부 네트워크 설정까지 필요합니다.

참고자료: [Openshift](https://developers.redhat.com/learn/openshift/deploy-openshift-data-foundation-across-availability-zones-using-multus)](<https://developers.redhat.com/learn/openshift/deploy-openshift-data-foundation-across-availability-zones-using-multus)



## 2. 용어 정리


| 용어                          | 정의                                | 역할                         |
| --------------------------- | --------------------------------- | -------------------------- |
| Primary Network             | Pod의 기본 네트워크                      | eth0과 Pod IP 부여            |
| Secondary Network           | 추가로 붙는 네트워크                       | net1, net2 순서              |
| Meta-plugin                 | 직접 인터페이스를 안 만들고 다른 플러그인을 부르는 플러그인 | **Multus**                 |
| Delegate                    | Meta-plugin이 대신 호출하는 플러그인         | kindnet, Calico, macvlan 등 |
| NetworkAttachmentDefinition | Secondary Network 설정을 담는 CRD      | 줄여서 NAD. 안에 CNI 설정이 들어감    |
| PF / VF                     | SR-IOV NIC의 실제 장치 / 쪼개진 가상 장치     | VF를 Pod에 통째로 넘김            |




## 3. Chaining만으로 여러 네트워크를 붙일 수 있나

CNI 플러그인을 여러 개 이어서 쓰는 chaining이 이미 있는데, Pod에 네트워크를 여러 개 붙일 때도 이걸 쓰면 되지 않을까 생각했음. 그런데 플러그인을 여러 개 실행하는 것과 네트워크 연결을 여러 개 만드는 건 다른 일임.

예컨대 W1 실습의 kindnet 설정은 ptp + host-local + portmap이었음. ptp가 host-local에 IP를 요청해서 eth0 연결을 만들고, 그다음 portmap이 포트 매핑 규칙을 추가함. 여기서 체인은 ptp → portmap이고, 주소 할당은 ptp가 내부에서 host-local을 호출해서 맡김.

```text
eth0 연결 요청 → ptp가 연결 생성 → portmap이 포트 매핑 추가
```

이렇게 하나의 연결을 설정하는 작업을 나눠서 순서대로 처리하는 게 **chaining**임. 설정의 plugins 배열 순서대로 실행하고, 뒤 플러그인은 앞 플러그인의 결과(prevResult)를 받아서 이어서 작업함. W3에서 meta로 분류한 bandwidth를 붙이면 대역폭을 제한하고, tuning을 붙이면 MTU나 sysctl 같은 설정을 조정할 수 있음. 지울 때는 역순으로 실행해서 나중에 얹은 것부터 걷어냄.

체인 안의 플러그인은 전부 같은 CNI\_IFNAME(보통 eth0)을 받음. 그래서 뒤에 플러그인을 더 적는다고 별도 네트워크에 연결되는 net1이 자동으로 생기는 건 아님. eth0과 net1을 각각 붙이려면 네트워크마다 설정을 고르고, 인터페이스 이름을 다르게 줘서 별도로 호출해야 함. 그 역할을 Multus가 맡음. Multus가 호출하는 네트워크 설정 안에도 체인을 쓸 수 있음.

참고자료: [CNI SPEC](https://www.cni.dev/docs/spec/), [Multus quickstart](https://k8snetworkplumbingwg.github.io/multus-cni/docs/quickstart.html)



## 4. Multus는 CNI를 감싸는 구조

### 4.1 호출 흐름

런타임 입장에서 Multus는 그냥 CNI 플러그인 하나. 런타임은 원래대로 CNI를 한 번 부르고, Multus가 그 안에서 다른 플러그인들을 차례로 대신 호출함.

```text
kubelet → CRI → containerd → CNI 호출: multus
                                ├─ delegate 1: Primary CNI → eth0, Pod IP
                                └─ delegate 2: NAD에 적힌 macvlan → net1
```

첫 번째로 호출한 네트워크가 Pod IP를 주고, Pod 어노테이션으로 요청한 나머지가 net1, net2로 붙음.

설치하면 Multus가 /etc/cni/net.d에 있던 첫 번째 설정을 읽어서 00-multus.conf를 만듦. Primary CNI를 갈아끼우는 게 아니라 그 앞에 서서 감싸는 구조. 그래서 Multus가 Primary CNI보다 먼저 뜨면 감쌀 설정이 없어서 Pod 생성이 계속 실패할 수 있음. readinessindicatorfile에 Primary 설정 파일 경로를 적어두면 그 파일이 생길 때까지 기다림.

### 4.2 thin과 thick


| 구분    | 처리 구조                                                                                                                |
| ----- | -------------------------------------------------------------------------------------------------------------------- |
| thin  | 노드의 multus 바이너리가 전부 처리 / 런타임 → multus → Primary CNI / 추가 CNI                                                         |
| thick | 노드의 multus-shim이 unix socket으로 multus-daemon Pod에 요청을 넘김 / 런타임 → multus-shim → multus-daemon → Primary CNI or 추가 CNI |


thick은 W5에서 본 cilium-cni랑 cilium-agent랑 같은 모양. 노드에 깔리는 바이너리는 얇게 두고, NAD를 조회하고 delegate를 부르는 실제 작업은 DaemonSet Pod가 맡음.

참고자료: [Multus configuration](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/configuration.md), [Multus thick plugin](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/thick-plugin.md)



## 5. Secondary 인터페이스는 Kubernetes 바깥

### 5.1 선언과 요청

NAD는 Secondary Network의 CNI 설정을 API 객체로 들고 있는 CRD. spec.config 안에 CNI 설정 JSON이 그대로 들어감. W3에서 본 /etc/cni/net.d 파일과 내용은 같은데, 노드마다 파일을 깔지 않고 API로 관리한다는 점이 다름.

Pod는 어노테이션에 NAD 이름을 적어서 요청함.

```yaml
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-conf
```

인터페이스 이름을 따로 안 주면 net1부터 붙고, macvlan-conf@data0처럼 적으면 이름을 정할 수 있음. 실제로 무엇이 붙었는지는 Multus가 network-status 어노테이션에 적어둠.

### 5.2 Kubernetes 관리 외 영역

kubectl get pod -o wide의 IP 칸에는 eth0 IP만 나옴. net1 IP는 Multus가 network-status 어노테이션에 따로 기록함. 이 정보가 기록되어 있다고 Service나 NetworkPolicy 같은 기본 기능이 추가 연결까지 자동으로 관리하는 건 아님.


| 기능                       | eth0 (Primary)  | net1 (Secondary)             |
| ------------------------ | --------------- | ---------------------------- |
| Service, EndpointSlice   | O               | 자동 등록 X                      |
| Kubernetes NetworkPolicy | Primary CNI가 구현하면 적용 | 자동 적용 보장 X / 추가망 정책 구현 필요 |
| IPAM                     | Primary CNI     | NAD마다 따로                     |


W2에서 정리한 네트워크 모델은 Pod끼리 그 IP로 통신되게 하라는 요구였는데, 이건 Primary에 대한 약속. Secondary는 이 약속 바깥에 있어서 정책과 주소, 경로를 직접 챙겨야 함.

**정책**: Kubernetes NetworkPolicy는 레이블로 Pod를 선택하고, 실제 차단은 CNI 같은 네트워크 구현이 맡음. macvlan이나 ipvlan으로 붙인 net1에도 기존 정책이 자동으로 적용된다고 가정하면 안 됨. 추가망에도 정책이 필요하면 그 연결을 지원하는 구현을 따로 마련해야 함. multi-networkpolicy 프로젝트가 NetworkPolicy와 같은 형식의 MultiNetworkPolicy CRD를 제공하고, 구현체로 iptables 버전과 tc 버전(Mellanox)이 있음.

**주소**: NAD에 host-local을 쓰면 W3에서 본 대로 노드 로컬 파일로 주소를 관리함. 노드끼리 서로의 할당 상태를 모르니까, 같은 NAD를 여러 노드가 쓰면 같은 IP를 줄 수 있음. 그래서 클러스터 단위로 관리하는 whereabouts 같은 IPAM이나 외부 DHCP를 씀.

**경로**: 패킷이 eth0과 net1 중 어디로 나갈지는 W1에서 본 대로 Pod 안 라우팅 테이블이 정함. 보통은 net1 서브넷만 net1으로 직결되고, 나머지는 eth0의 기본 경로를 탐. 어노테이션에 default-route를 주면 기본 경로를 net1 쪽으로 돌릴 수 있는데, 이러면 ClusterIP로 가는 트래픽까지 net1으로 나갈 수 있어서 주의 필요.

참고자료: [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/), [multi-networkpolicy](https://github.com/k8snetworkplumbingwg/multi-networkpolicy)



## 6. Multus 선택지: macvlan, ipvlan, SR-IOV

W3 표에서 macvlan을 main 플러그인으로 분류했음. 이번에 보는 셋은 모두 노드의 물리 인터페이스를 쪼개서 Pod에 바로 붙이는 방식이라, 지금까지 본 veth랑 bridge 경로를 거치지 않음.


| 구분      | macvlan                             | ipvlan          | SR-IOV         |
| ------- | ----------------------------------- | --------------- | -------------- |
| 쪼개는 주체  | 커널                                  | 커널              | NIC 하드웨어       |
| MAC     | 인터페이스마다 다름                          | master와 같음      | VF마다 다름        |
| DHCP    | 가능                                  | 기본 DHCP IPAM 미지원 | -              |
| 모드      | bridge(기본), private, vepa, passthru | l2(기본), l3, l3s | -              |
| 스케줄링 영향 | 없음                                  | 없음              | 있음. VF가 확장 리소스 |


macvlan은 쪼갠 인터페이스마다 MAC을 따로 줌. 물리망에서 보면 장치가 여러 대 붙은 것처럼 보여서 기존 DHCP 서버를 그대로 쓸 수 있음. ipvlan은 반대로 master의 MAC을 같이 쓰고 IP로 구분함. ClientID로 클라이언트를 구분하면 DHCP를 쓸 수 있지만, 현재 표준 DHCP IPAM 플러그인은 이를 지원하지 않음.

같은 master에 macvlan이랑 ipvlan을 같이 쓸 수는 없음. ipvlan 가상 인터페이스는 master와 직접 통신하지 못해서 호스트에 닿는 다른 연결이 필요함. 기존 eth0으로 호스트랑 통신할 수 있으면 그 연결을 쓰면 되고, 그런 연결이 없을 때 ptp 같은 인터페이스를 추가해야 함.

SR-IOV는 쪼개는 일을 커널이 아니라 NIC 하드웨어가 함. NIC 하나(PF)가 가상 장치 여러 개(VF)로 나뉘고, 그중 하나를 Pod에 통째로 넘김. 플러그인이 둘인데, Device Plugin이 노드의 VF를 확장 리소스로 알리고 SR-IOV CNI가 할당된 VF를 Pod netns로 옮김. Pod는 CPU처럼 resources에 VF를 요청하니까 남은 VF가 없는 노드에는 아예 스케줄되지 않음. macvlan이랑 ipvlan에는 없는 제약.

참고자료: [macvlan](https://www.cni.dev/plugins/current/main/macvlan/), [ipvlan](https://www.cni.dev/plugins/current/main/ipvlan/), [SR-IOV Network Device Plugin](https://github.com/k8snetworkplumbingwg/sriov-network-device-plugin)



## 7. 질문

1. macvlan, ipvlan, SR-IOV가 실제 네트워크 연결을 만드는 방식에 차이가 있다는 것은 이해했으나 해당 차이가 발생하며 어떤 장단점 혹은 특징이 생기는지?
2. default-route로 기본 경로를 net1로 돌렸을 때 ClusterIP 통신이 실제로 깨지는지? W5의 Cilium처럼 connect 시점에 주소를 바꾸는 방식이면 결과가 달라지는지?