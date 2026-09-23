# W4. Primary CNI

## 1. W3에서 이어지는 질문

W3에서는 CNI 규격이 호출 방식과 주고받는 값까지만 정하고, **노드를 넘어가는 방식은 구현체 자유**라는 걸 확인했습니다. 이번주는 그 빈칸을 대표 CNI 두 개가 실제로 어떻게 채우는지, 그리고 **같은 Pod-to-Pod 통신이 CNI에 따라 어떤 경로로 달라지는지**를 보려 합니다.

Flannel은 오버레이의 기본형이고 Calico는 underlay 라우팅의 기본형이라 둘을 나란히 놓으면 차이가 잘 보입니다. W3에서 논의해봤던 L2 브리지와 L3 라우팅의 디버깅 차이도 여기서 일부 답해봤습니다.

이번 주차의 메인은 주요 CNI가 패킷을 어떻게 처리하는지입니다. 데이터 플레인과 NetworkPolicy 기준으로 다시 놓아보면:


| CNI     | 데이터 플레인                               | NetworkPolicy          |
| ------- | ------------------------------------- | ---------------------- |
| KindNet | ptp + host-local, 노드별 podCIDR 경로      | 공식 문서에 언급 없음           |
| Flannel | VXLAN(기본), host-gw, WireGuard, IPIP 등 | X (flanneld 자체는 미지원)   |
| Calico  | BGP L3 라우팅, IPIP/VXLAN, eBPF 옵션       | O                      |
| Antrea  | OVS 브리지, 기본은 Encap(터널) 모드             | O (OVS 플로우로 적용)        |
| Cilium  | eBPF, 오버레이(VXLAN/Geneve) 또는 네이티브 라우팅  | O (Identity 기반, L7 포함) |


참고자료: [kind 설정](https://kind.sigs.k8s.io/docs/user/configuration/), [kindnetd](https://github.com/kubernetes-sigs/kind/blob/main/images/kindnetd/README.md), [Antrea 아키텍처](https://antrea.io/docs/main/docs/design/architecture/), [Cilium 소개](https://docs.cilium.io/en/stable/overview/intro/)



## 2. 용어 정리


| 용어                     | 정의                                    | 역할                            |
| ---------------------- | ------------------------------------- | ----------------------------- |
| Overlay                | 다른 네트워크 위에 한 겹 더 얹은 네트워크, 패킷을 감싸서 보냄  | 하부 인프라가 Pod IP를 몰라도 되게 함      |
| Underlay (non-overlay) | 하부 네트워크 위에서 감싸지 않고 바로 동작              | 하부 네트워크가 Pod IP를 알아야 함        |
| VXLAN                  | L2 프레임을 UDP로 감싸서 보내는 터널 규격 (RFC 7348) | 노드 간 Pod 트래픽을 감쌈              |
| VTEP                   | VXLAN Tunnel End Point                | VXLAN 터널을 시작하고 끝내는 지점         |
| VNI                    | VXLAN Network Identifier, 24비트        | 가상 세그먼트를 최대 약 1,600만 개까지 구분   |
| IPIP                   | IP 패킷을 IP 헤더로 한 번 더 감쌈                | 헤더 20바이트로 VXLAN보다 작음, IPv4 전용 |
| MTU                    | 인터페이스가 한 번에 보낼 수 있는 최대 패킷 크기          | 캡슐화하면 헤더만큼 줄여야 함              |
| BGP / AS               | L3 경로 교환 프로토콜 / 경로를 교환하는 단위           | 노드끼리, 또는 노드와 물리 라우터가 경로를 주고받음 |
| Route Reflector        | BGP 클라이언트들이 연결하는 중앙 지점                | 노드끼리 전부 연결하는 full mesh를 대체    |


VXLAN 표준 포트는 IANA가 배정한 **UDP 4789**이고, RFC에는 "VTEP는 VXLAN 패킷을 조각내면 안 된다(MUST NOT fragment)"고 적혀 있습니다. 뒤에서 5.1절 MTU 이야기와 이어지는 부분.

참고자료: [VXLAN RFC 7348](https://datatracker.ietf.org/doc/html/rfc7348), [Calico 네트워킹 방식 선택](https://docs.tigera.io/calico/latest/networking/determine-best-networking)



## 3. Flannel: 주소를 나눠주고 호스트 사이를 잇기

### 3.1 구성 요소

Flannel 문서는 스스로를 Kubernetes용 L3 네트워크 패브릭이라고 소개합니다. 책임 범위를 한 줄로 긋는 문장이 있는데, **컨테이너가 호스트에 어떻게 연결되는지는 관여하지 않고 호스트 사이에 트래픽을 어떻게 나르는지만** 다룬다는 것입니다.


| 구성                        | 위치               | 역할                                                          |
| ------------------------- | ---------------- | ----------------------------------------------------------- |
| flanneld                  | 노드마다 하나          | 전체 주소 공간에서 노드별 서브넷(lease)을 배정, 백엔드로 호스트 간 전달.               |
| `--kube-subnet-mgr`       | flanneld 옵션      | etcd 대신 Kubernetes API에서 서브넷 배정을 받음.                        |
| `/run/flannel/subnet.env` | 노드 파일            | `FLANNEL_NETWORK`, `FLANNEL_SUBNET`, `FLANNEL_MTU` 등을 기록.   |
| flannel CNI 플러그인          | `/opt/cni/bin`   | subnet.env를 읽고 실제 작업은 다른 플러그인에 넘김. (기본 bridge + host-local) |
| `cni0`                    | 노드의 Linux bridge | bridge 플러그인의 기본 이름. 같은 노드 컨테이너를 가상 스위치에 꽂음.                 |


그래서 **같은 노드 안의 연결은 표준 bridge 플러그인이, 노드 사이 전달은 flanneld 백엔드가** 맡는 구조가 됩니다. W3에서 본 main / ipam 플러그인 분리가 Flannel에서는 위임(delegate)으로 나타납니다.

참고자료: [Flannel README](https://github.com/flannel-io/flannel), [flannel CNI 플러그인](https://github.com/flannel-io/cni-plugin), [Flannel 설정](https://github.com/flannel-io/flannel/blob/master/Documentation/configuration.md), [CNI bridge 플러그인](https://www.cni.dev/plugins/current/main/bridge/)

### 3.2 백엔드


| 백엔드               | 방식                                    | 조건 / 비고                                |
| ----------------- | ------------------------------------- | -------------------------------------- |
| vxlan (기본값, 권장)   | 커널 VXLAN으로 캡슐화                        | 커널이 UDP 8472 사용                        |
| host-gw           | 다른 노드 서브넷으로 가는 IP 경로를 원격 노드 IP로 직접 만듦 | 노드끼리 L2로 직결돼 있어야 함                     |
| wireguard / ipsec | 커널 기능으로 캡슐화 + 암호화                     |                                        |
| ipip              | 커널 IPIP로 캡슐화                          | 오버헤드가 가장 작음, IPv4 전용                   |
| udp               | 디버깅 전용                                | UDP 8285 사용, VXLAN·host-gw를 못 쓰는 네트워크용 |


기본이 왜 VXLAN인지는 조건 열에서 읽힙니다. host-gw는 **호스트 사이 L2 직결이 필요**하고, VXLAN은 노드끼리 UDP만 닿으면 됩니다.

리눅스에서 Flannel VXLAN은 RFC 표준 4789가 아니라 **커널 기본값 8472**를 씁니다(윈도우는 4789 필수). 노드 사이 방화벽을 열 때 헷갈리기 쉬운 부분.

참고자료: [Flannel backends](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md), [Flannel troubleshooting](https://github.com/flannel-io/flannel/blob/master/Documentation/troubleshooting.md)

### 3.3 Flannel이 하지 않는 것

README에 따르면 flanneld 바이너리는 **NetworkPolicy를 자체적으로 적용하지 않습니다.** 정책이 필요하면 Kubernetes SIGs의 컨트롤러나 Calico, Cilium 같은 서드파티를 붙이는 방식을 안내합니다. 



## 4. Calico: 경로를 광고해서 잇기

### 4.1 구성 요소


| 구성                   | 역할                                                   |
| -------------------- | ---------------------------------------------------- |
| Felix                | 호스트에 경로와 ACL(정책 규칙) 등 연결에 필요한 설정을 씀                  |
| BIRD                 | Felix가 만든 경로를 받아 BGP 피어에게 배포                         |
| confd                | 데이터스토어의 BGP 설정 변경을 보고 BIRD 설정 파일을 다시 생성              |
| Typha                | 데이터스토어와 Felix 사이에서 상태를 캐시하고 이벤트 중복을 줄여 데이터스토어 부하를 낮춤 |
| kube-controllers     | Kubernetes API를 보고 정책, 네임스페이스, 노드 컨트롤러로 동작           |
| CNI 플러그인 / IPAM 플러그인 | Kubernetes에 Calico 네트워킹 제공 / IP Pool 기준으로 Pod 주소 할당  |


W3 5절에서 본 **노드별 에이전트 + 조정 컨트롤러** 구조가 그대로입니다. Felix는 iptables, nftables, eBPF 중 하나를 데이터 플레인으로 쓸 수 있습니다.

참고자료: [Calico 컴포넌트 아키텍처](https://docs.tigera.io/calico/latest/reference/architecture/overview), [FelixConfiguration](https://docs.tigera.io/calico/latest/reference/resources/felixconfig)

### 4.2 BGP로 경로 광고

BGP를 켜면 기본값은 모든 노드가 서로 iBGP로 연결되는 **full mesh**이고, AS 번호는 노드별로 따로 지정하지 않으면 **64512**입니다. 문서는 대략 100노드 이하에서는 full mesh를, 그보다 크면 Route Reflector를 권장합니다. full mesh는 노드가 n개면 연결이 n(n-1)/2개라서 규모가 커지면 비효율적이기 때문.

BGP가 없으면 노드 A는 노드 B에 있는 Pod 대역이 어디 있는지 모르고 어디로 향해야 하는지 모릅니다. 더 나아가 ToR(Top of Rack) 라우터와 피어링하면 Pod IP가 클러스터 밖 네트워크에서도 라우팅되는 non-overlay 구성이 됩니다.

참고자료: [Calico BGP 설정](https://docs.tigera.io/calico/latest/networking/configuring/bgp)

### 4.3 캡슐화 모드: 언제 오버레이로 떨어지나

Calico도 필요하면 감쌉니다. IP Pool마다 IPIP 또는 VXLAN 모드를 고릅니다.


| 모드          | 동작                 | 쓰는 경우                        |
| ----------- | ------------------ | ---------------------------- |
| Never       | 감싸지 않음             | 하부 네트워크가 Pod IP를 라우팅할 수 있을 때 |
| CrossSubnet | 서브넷 경계를 넘는 트래픽만 감쌈 | 같은 서브넷 통신은 오버헤드 없이 두고 싶을 때   |
| Always      | 노드 간 트래픽을 전부 감쌈    | 하부 네트워크가 Pod IP를 전혀 모를 때     |


그럼 언제 캡슐화로 넘어가나? 문서 표현 그대로 워크로드 IP를 쉽게 알게 할 수 없는 하부 네트워크 위에서 돌릴 때입니다. 예시로 여러 VPC, 서브넷에 걸친 AWS, BGP 피어링이 안 되는 퍼블릭 클라우드를 듭니다. Calico의 권장은 가능하면 캡슐화 없이 돌리는 것.

참고자료: [Calico overlay 설정](https://docs.tigera.io/calico/latest/networking/configuring/vxlan-ipip)

### 4.4 Pod 쪽 모양: bridge가 없음

Calico FAQ에 따르면 **각 호스트가 자기 워크로드의 게이트웨이 라우터** 역할을 하고, Pod마다 /32 경로를 받는 완전한 L3 구조라 워크로드 수준에는 브로드캐스트 도메인이 없습니다. 

Pod 안의 게이트웨이는 `169.254.1.1`로 잡히는데, 문서는 이 주소를 Calico 라우터의 주소로 씁니다. 링크 로컬 주소를 써서 IP 공간을 아끼고 사용자가 게이트웨이를 따로 설정하지 않아도 되게 하는 것. Pod의 경로 테이블이 모든 트래픽을 이 게이트웨이로 보내기 때문에 Pod가 ARP로 찾는 주소는 이것 하나뿐이고, Calico는 **이 주소를 해석할 때만 proxy ARP를 씁니다.** W1 6절 기준으로 보면, 이게 없으면 Pod는 next hop의 MAC을 못 찾아서 밖으로 못 나갑니다.

노드 쪽에서 Pod와 연결된 인터페이스는 이름 접두사로 구분합니다. FelixConfiguration의 `interfacePrefix` 기본값이 `cali`.

참고자료: [Calico FAQ](https://docs.tigera.io/calico/latest/reference/faq)



## 5. Overlay vs Underlay


| 방식                   | 장점                                            | 비용                                  |
| -------------------- | --------------------------------------------- | ----------------------------------- |
| L2 브리지 (OVS, Antrea) | 정책과 Service 부하분산을 OVS 안에서 처리                  | Antrea도 기본은 터널이라 오버레이 비용은 동일        |
| L3 라우팅 (BGP, Calico) | 캡슐화 없음, ToR 라우터와 피어링하면 Pod IP가 클러스터 밖에서도 라우팅됨 | 하부 네트워크가 Pod IP를 알아야 함              |
| 오버레이 (VXLAN/IPIP)    | 하부 인프라를 거의 안 건드리고 어디서든 동작                     | 캡슐화에 CPU를 조금 쓰고, 헤더만큼 안쪽 패킷 크기가 줄어듦 |
| eBPF                 | W5(Cilium)에서 확인                               | W5에서 확인                             |


참고자료: [Calico 네트워킹 방식 선택](https://docs.tigera.io/calico/latest/networking/determine-best-networking), [Antrea 아키텍처](https://antrea.io/docs/main/docs/design/architecture/)

### 5.1 MTU

오버레이 비용 중 안쪽 패킷 크기가 줄어든다는 게 MTU 이야기입니다. Calico 문서 기준으로 방식별 헤더 크기를 정리하면:


| 방식               | 헤더 오버헤드 | 하부 MTU 1500일 때 권장 MTU |
| ---------------- | ------- | --------------------- |
| IPIP             | 20      | 1480                  |
| VXLAN (IPv4)     | 50      | 1450                  |
| VXLAN (IPv6)     | 70      | 1430                  |
| WireGuard (IPv4) | 60      | 1440                  |
| WireGuard (IPv6) | 80      | 1420                  |


Flannel 문서의 subnet.env 예시에도 `FLANNEL_MTU=1450`이 들어 있습니다. VXLAN 기본 백엔드와 같은 숫자.

MTU를 너무 크게 잡았을 때 문서가 언급하는 증상은 **패킷 손실과 단편화 문제**입니다. RFC 7348도 VTEP는 조각내지 말아야 한다고 정해두고, 캡슐화된 더 큰 프레임을 수용하도록 네트워크 MTU를 설정하거나 Path MTU discovery를 쓰라고 권합니다. Flannel troubleshooting 문서는 MTU를 의심할 때 확인 순서를 ① 네트워크 인터페이스 MTU → ② subnet.env의 MTU → ③ 컨테이너 veth MTU로 제시합니다.

참고자료: [Calico MTU 설정](https://docs.tigera.io/calico/latest/networking/configuring/mtu), [Flannel troubleshooting](https://github.com/flannel-io/flannel/blob/master/Documentation/troubleshooting.md)

### 5.2 Pod IP가 클러스터 밖에서 보인다는 것

Calico 문서 기준으로 정리하면:

- Pod IP가 클러스터 밖으로 라우팅되지 **않는** 구성이면, Pod가 클러스터 밖으로 나갈 때 Kubernetes가 SNAT로 출발지 IP를 노드 IP로 바꿈
- 라우팅**되는** 구성(ToR과 BGP 피어링 등)이면 Pod IP가 더 넓은 네트워크 전체에서 **고유해야 함.** 클러스터가 여러 개면 클러스터마다 다른 Pod CIDR을 써야 함

즉 underlay 방식은 Pod IP를 네트워크의 정식 구성원으로 만드는 대신, 주소 계획을 클러스터 밖 네트워크와 같이 맞춰야 합니다.

참고자료: [Calico 네트워킹 방식 선택](https://docs.tigera.io/calico/latest/networking/determine-best-networking)



## 6. Pod-to-Pod 패킷 흐름 비교

### 6.1 같은 노드


| CNI     | 경로                                                   | 근거              |
| ------- | ---------------------------------------------------- | --------------- |
| Flannel | Pod A → veth → `cni0` bridge → veth → Pod B          | bridge 플러그인에 위임 |
| Calico  | Pod A → 게이트웨이 169.254.1.1(호스트) → 호스트의 /32 경로 → Pod B | 호스트가 게이트웨이 라우터  |


같은 노드인데도 Flannel은 가상 스위치가 넘기고, Calico는 호스트가 라우터로서 넘깁니다.

### 6.2 다른 노드 — Flannel VXLAN

```text
Pod A → veth → cni0 → 노드 A
                         ↓
       flannel VXLAN 백엔드: 원래 프레임을 VXLAN 헤더 + UDP(8472)로 감쌈
       바깥 IP 헤더 = 노드 A의 VTEP → 노드 B의 VTEP
                         ↓
       하부 네트워크 (노드 IP끼리의 UDP로만 보임)
                         ↓
       노드 B에서 VXLAN 해제 → cni0 → veth → Pod B
```

### 6.3 다른 노드 — Calico BGP (캡슐화 없음)

```text
Pod A → 게이트웨이 169.254.1.1 (proxy ARP로 호스트가 응답) → 노드 A 경로 조회
                         ↓
       BGP로 받아둔 경로: 노드 B의 블록 → 노드 B
       (블록 단위로 묶어서 광고하므로 Pod마다 경로가 생기지 않음, 8절)
                         ↓
       하부 네트워크 (원본 Pod IP 그대로)
                         ↓
       노드 B의 /32 경로 → cali 인터페이스 → Pod B
```

### 6.4 디버깅할 때 먼저 보는 것

문서가 직접 안내하는 확인 지점만 모으면:


| 확인 지점   | Flannel                                                    | Calico                                            |
| ------- | ---------------------------------------------------------- | ------------------------------------------------- |
| 노드 간 연결 | 방화벽에서 UDP 8472(vxlan) / 8285(udp) 허용 여부                    | `calicoctl node status`로 BGP 피어 상태(`Established`) |
| 주소 배정   | `kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'` | IP Pool과 블록                                       |
| MTU     | 인터페이스 → subnet.env → 컨테이너 veth 순서                          | 모드별 권장 MTU 표 (5.1)                                |
| 로그      | `kubectl logs -n kube-flannel <pod> -c kube-flannel`       | —                                                 |


오버레이는 터널이 통하는지(포트, MTU)를, BGP는 **피어링이 맺어져 경로가 교환되는지**를 먼저 봅니다.

참고자료: [Flannel troubleshooting](https://github.com/flannel-io/flannel/blob/master/Documentation/troubleshooting.md), [calicoctl node status](https://docs.tigera.io/calico/latest/reference/calicoctl/node/status)



## 7. NetworkPolicy는 어느 계층에서 처리되나

Kubernetes 문서 기준:

- 기본값은 **비격리.** ingress·egress 모두 정책이 없으면 전부 허용
- **NetworkPolicy를 구현하는 컨트롤러 없이 리소스만 만들면 아무 효과가 없음**
- 다루는 범위는 IP·포트 수준(L3/L4)

참고자료: [NetworkPolicy 문서](https://kubernetes.io/docs/concepts/services-networking/network-policies/)


| CNI     | 정책 적용                                          | 적용 방식                              |
| ------- | ---------------------------------------------- | ---------------------------------- |
| Flannel | X (flanneld 자체는 미지원)                           | 다른 프로젝트를 붙여야 함                     |
| Calico  | O (Kubernetes NetworkPolicy와 함께 또는 Calico 정책만) | Felix가 호스트에 ACL로 씀                 |
| Antrea  | O                                              | Controller가 계산, Agent가 OVS 플로우로 적용 |


그래서 Flannel에 NetworkPolicy가 없으면 실제로 무엇이 안 되나? 정책 리소스를 만들어도 **효과가 없고, 모든 트래픽이 기본값대로 허용된 상태로 남습니다.**

Calico는 Kubernetes NetworkPolicy 외에 자체 리소스를 둡니다. 네임스페이스 단위의 `NetworkPolicy`와, 네임스페이스와 상관없이 Pod, VM, 호스트 인터페이스에 걸 수 있는 `GlobalNetworkPolicy`. L5\~7 조건은 **Istio와 함께 쓸 때만** 지원한다고 명시돼 있습니다. Cilium은 HTTP 메서드, 경로 단위 L7 규칙을 자체로 지원한다고 소개하는데, 이건 다음주에 공부하며 확인해볼 수 있을 것 같습니다.

Flannel을 쓰면서 정책이 필요하면 Calico 문서가 안내하는 조합(Canal)이 있습니다. 문서 표 기준으로 **정책은 Calico, 오버레이는 flannel VXLAN, IPAM은 host-local**. 다만 문서는 지금은 Calico 자체 VXLAN을 쓰는 쪽을 단순함 때문에 권장합니다.

참고자료: [Calico 네트워크 정책](https://docs.tigera.io/calico/latest/network-policy/get-started/calico-policy/calico-network-policy), [Calico for policy + flannel for networking](https://docs.tigera.io/calico/latest/getting-started/kubernetes/flannel/install-for-flannel)



## 8. IPAM은 어느 계층에서 처리되나

W3 4절에서는 IPAM을 왜 분리했는지까지 정리했습니다. 이번에는 각도를 바꿔서 **어느 계층에서 처리되는지**를 봅니다. Flannel(과 KindNet)은 host-local, Calico는 calico-ipam을 쓰는데, 누가 범위를 정하고 누가 주소를 주고 상태를 어디에 두는지 단계별로 나누면:


| 단계         | Flannel (host-local)                                          | Calico (calico-ipam)                                   |
| ---------- | ------------------------------------------------------------- | ------------------------------------------------------ |
| 전체 범위      | net-conf의 `Network` (Pod CIDR과 일치해야 함)                        | IP Pool (기본은 Pod CIDR 전체를 Pool 하나로)                    |
| 노드별 범위     | kube-subnet-mgr 모드에서는 Kubernetes API가 배정한 `node.spec.podCIDR` | 노드에 연결된 블록 (기본 /26 = 주소 64개)                           |
| 노드별 크기 기본값 | kube-controller-manager `--node-cidr-mask-size` 기본 IPv4 /24   | 블록 크기는 IP Pool마다 조정 가능                                 |
| 개별 Pod 주소  | host-local이 노드 파일시스템에 기록 (`/var/lib/cni/networks/<네트워크명>`)    | calico-ipam이 Calico 데이터스토어(Kubernetes API 또는 etcd)에 기록 |
| 겹치지 않음 보장  | 한 노드 안에서만                                                     | 클러스터 전체 IP Pool 기준                                     |
| 노드 범위가 차면  | 문서에 명시 없음 (질문 3)                                              | 필요하면 새 블록을 만들고, 노드에 연결되지 않은 블록의 주소도 줄 수 있음             |


Calico가 블록으로 나누는 이유는 "같은 노드 Pod의 주소를 효율적으로 묶어서 라우팅 테이블 크기를 줄이기 위해서"입니다.

host-local은 문서에 따르면 **한 호스트 안에서만** 주소가 겹치지 않게 보장합니다. 노드끼리 겹치지 않는 건 노드마다 다른 podCIDR을 받는 상위 단계(컨트롤 플레인)가 맡는 구조.

참고자료: [Calico IP 주소 관리](https://docs.tigera.io/calico/latest/networking/ipam/get-started-ip-addresses), [Calico 블록 크기](https://docs.tigera.io/calico/latest/networking/ipam/change-block-size), [host-local 플러그인](https://www.cni.dev/plugins/current/ipam/host-local/), [kube-controller-manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)



## 9. 실습

이번 주차는 시간관계상 실습을 진행하지 못했습니다. W1에서 쓴 예측 → 조작 → 관찰 → 복구 순서를 CNI 교체에 그대로 적용할 수 있을 것 같아서, kind 클러스터에 Calico를 올린다면 확인할 지점만 적어두고 이후 주차에 이어서 해볼 예정입니다.


| 확인 대상            | 볼 것                | 예측                               |
| ---------------- | ------------------ | -------------------------------- |
| `/etc/cni/net.d` | 설정 파일이 무엇으로 바뀌었는지  | kindnet 설정 자리에 Calico 설정         |
| `/opt/cni/bin`   | 어떤 바이너리가 추가됐는지     | calico, calico-ipam              |
| Pod의 경로 테이블      | ptp인지, 게이트웨이가 무엇인지 | 169.254.1.1 게이트웨이                |
| 노드의 경로 테이블       | 다른 노드의 Pod 대역 경로   | BGP로 받은 블록 단위 경로                 |
| 기존 Pod           | 예전 구성을 그대로 들고 있는지  | CNI는 Pod 생성 시 한 번 호출되므로 재생성해야 바뀜 |


마지막 줄을 제일 확인해보고 싶은데, CNI를 바꿔도 이미 떠 있는 Pod는 예전 네트워크 구성을 그대로 들고 있을 것으로 예상합니다.

참고자료: [kind 설정 (`disableDefaultCNI`)](https://kind.sigs.k8s.io/docs/user/configuration/)



## 10. 질문

1. Calico CrossSubnet 모드에서는 같은 서브넷 통신은 안 감싸고 넘어갈 때만 감싸는데, 그러면 Pod MTU는 어느 쪽 기준으로 잡아야 하는지? 작은 쪽에 맞추면 같은 서브넷 통신이 손해를 보는 건지?
2. Flannel에서 NetworkPolicy가 효과 없이 남아 있는 것처럼, 운영 중에 정책이 실제로 적용되고 있는지는 어떻게 확인할 수 있는지?
3. host-local을 쓰는 Flannel에서 노드의 podCIDR(/24)이 다 차면 새 Pod는 어느 단계에서 어떤 에러로 실패하는지?
4. flanneld나 calico-node가 죽었을 때, 이미 커널에 들어간 경로로 기존 통신은 계속되는지? 노드가 추가되거나 Pod가 재생성되면 어느 시점부터 깨지는지?