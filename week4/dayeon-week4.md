
## Underlay vs Overlay 재정리
### Underlay
== Node가 연결된 실제 네트워크

```
Physical / Cloud Network
 ├── Switch
 ├── Router
 ├── VLAN
 └── VPC / VNet
```
Underlay는 기본적으로 Node 간 IP Reachability를 제공한다.

### Overlay

== 기존 Underlay 위에 별도의 논리적인 Pod Network를 구성
```
Pod Network
    ↓
Overlay
    ↓
Underlay
    ↓
Physical Network
```
Node 간 Packet을 보낼 때 Pod Packet을 Node Packet 안에 캡슐화할 수 있다.
```
Outer Packet
Source: Node A
Destination: Node B

    └── Inner Packet
        Source: Pod A
        Destination: Pod B
```
따라서 Underlay는 Pod IP를 알 필요 없이 Node A → Node B만 전달하면 된다.


## Routing과 BGP

Native Routing에서 중요한 것:

> 누가 Pod Network Route를 Underlay에 알려주는가?

Calico에서는 BGP를 이용해 Node 간 또는 Node와 외부 Router 사이에서 Pod Network Route를 교환할 수 있다.

```
Calico Node A
Pod CIDR: 10.244.1.0/24
       │
       │ BGP
       ▼
ToR / Router
       │
       │ BGP
       ▼
Calico Node B
Pod CIDR: 10.244.2.0/24
```

여기서 BGP는 Packet을 직접 전달하는 **프로토콜이 아니라 Routing Information을 교환하는 Control Plane이다**.

```
BGP
 ↓
Route Information
 ↓
Routing Table / RIB(routing info base) => control plane 
 ↓
FIB(forwarding info base) => data plane
 ↓
Packet Forwarding
```
Cisco 자료에서도 Calico Node를 L3 virtual router처럼 보고, BGP가 routing information을 교환하며 Linux kernel network stack이 실제 dataplane을 담당하는 구조로 설명한다.

<details>
  <summary>이론이 와닿지 않아서 아래와 같은 핸즈온을 했습니다만 단계를 참고하시라고 복.붙..했습니다</summary>

```text
                    "관찰"의 레벨

Whisker / Goldmane
    ↓
Pod A → Pod B
정책 ALLOW/DENY
protocol / ports
packet count / byte count
    │
    │  여기까지는 편하게 볼 수 있음
    ▼
────────────────────────────────
    │
    │  실제 packet / interface 관찰
    ▼
tcpdump
    ↓
caliXXXX
vxlan.calico
eth0
    ↓
Inner IP packet
Outer IP packet
VXLAN UDP 4789
VNI
    ↓
실제 wire-level 흐름
```

Hubble은 **Cilium의 observability 계층**이라 Cilium이 관리하는 datapath에서 flow를 보여준다. 기본적으로 L3/L4 flow visibility를 제공하고, L7 visibility는 별도 L7 proxy 기능을 사용할 때 가능하다. 즉 `Node A eth0에서 실제 VXLAN UDP 4789 packet이 어떻게 나갔는가` 같은 것은 Hubble의 주 목적이 아니다.

Calico에서는 현재 **Whisker + Goldmane**이 비슷한 역할을 한다. 다만 Calico flow log도 **개별 packet 하나하나를 raw packet으로 기록하는 게 아니라 connection data를 aggregation**해서 보여준다. source/destination, policy, packet/byte count 등을 볼 수 있다. 

그래서 네가 지금 공부하는 Week 4의 목적에는 오히려:

> **Whisker + `ip route` + `ip -d link` + `bridge fdb` + `tcpdump`**

이 조합이 아주 좋다.

---

# 1. 지금 `calico-lab`에서 가장 먼저 해볼 것

네 클러스터가 현재:

```text
IPPool
192.168.0.0/16
vxlanMode: CrossSubnet
ipipMode: Never
```

였지.

그런데 k3d는 **모든 cluster node를 같은 Docker network에 넣는다.** 따라서 두 Node가 같은 subnet에 있을 가능성이 높다. Calico의 `CrossSubnet`은 **destination node가 다른 subnet일 때만 VXLAN을 사용**한다. 따라서 지금 구성에서는 cross-node traffic을 발생시켜도 VXLAN이 안 보일 수 있다.

그래서 **실습 1에서는 VXLAN을 무조건 사용하도록 잠깐 바꾸자.**

```bash
kubectl config use-context k3d-calico-lab

kubectl patch ippool default-ipv4-ippool \
  --type=merge \
  -p '{"spec":{"vxlanMode":"Always","ipipMode":"Never"}}'
```

확인:

```bash
kubectl get ippool default-ipv4-ippool -o yaml
```

다음처럼 나오면 된다.

```yaml
spec:
  ipipMode: Never
  vxlanMode: Always
```

Calico는 `vxlanMode: Always`에서 Calico workload 간 traffic을 VXLAN으로 전송하도록 구성한다. 기본 VXLAN VNI는 4096이다. 

---

# 2. 두 Node에 Pod 하나씩 배치

먼저 Node 이름 확인.

```bash
kubectl get nodes -o wide
```

예를 들어:

```text
NAME                       STATUS   INTERNAL-IP
k3d-calico-lab-server-0   Ready    172.20.0.2
k3d-calico-lab-agent-0    Ready    172.20.0.3
```

환경마다 IP는 다르다.

그다음:

```bash
NODE1=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
NODE2=$(kubectl get nodes -o jsonpath='{.items[1].metadata.name}')

echo $NODE1
echo $NODE2
```

서버 Pod를 Node1에:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: calico-server
spec:
  nodeName: $NODE1
  containers:
  - name: nginx
    image: nginx:alpine
EOF
```

클라이언트 Pod를 Node2에:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: calico-client
spec:
  nodeName: $NODE2
  containers:
  - name: client
    image: nicolaka/netshoot:latest
    command: ["sleep", "3600"]
EOF
```

확인:

```bash
kubectl get pods -o wide
```

원하는 형태:

```text
calico-server   ...   192.168.x.x   k3d-calico-lab-server-0
calico-client   ...   192.168.y.y   k3d-calico-lab-agent-0
```

즉:

```text
Node 1
 └── Server Pod

Node 2
 └── Client Pod
```

---

# 3. Pod 내부에서 먼저 본다

Server IP를 변수로 저장.

```bash
SERVER_IP=$(kubectl get pod calico-server -o jsonpath='{.status.podIP}')
echo $SERVER_IP
```

Client에서 server로 요청:

```bash
kubectl exec calico-client -- curl -s http://$SERVER_IP
```

nginx HTML이 나오면 성공.

여기서 이미 중요한 사실 하나를 확인한 거야.

```text
Client Pod
  ↓
Pod IP
  ↓
???
  ↓
Server Pod IP
```

이제 `???`를 까보는 거다.

---

# 4. Pod 내부 Routing 확인

Client에서:

```bash
kubectl exec calico-client -- ip addr
```

그리고:

```bash
kubectl exec calico-client -- ip route
```

여기서 보는 건:

```text
eth0
default route
connected route
```

즉 Pod network namespace 관점에서

```text
Application
   ↓
Pod eth0
   ↓
Pod routing
```

을 확인하는 단계다.

---

# 5. Node에서 Calico interface 확인

이제 진짜 재미있는 부분.

k3d Node는 Docker container이기 때문에 host에서:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}'
```

를 실행하면 대략:

```text
k3d-calico-lab-server-0
k3d-calico-lab-agent-0
```

같은 container가 보일 거다.

Client가 있는 Node에 들어간다.

```bash
docker exec -it k3d-calico-lab-agent-0 sh
```

안에서:

```bash
ip -br link
```

그러면 대략:

```text
eth0
lo
caliXXXX
vxlan.calico
...
```

가 보일 것이다.

여기서 역할을 이렇게 생각하면 된다.

```text
Pod
 │
 │ veth
 ▼
caliXXXX
 │
 ▼
Linux Kernel
 │
 ├── Routing
 ├── Policy
 └── VXLAN
      │
      ▼
   eth0
```

Calico의 datapath도 기본적으로 Linux routing table과 firewall/dataplane을 중심으로 구성된다. 

---

# 6. Routing Table을 직접 본다

같은 Node에서:

```bash
ip route
```

특히:

```bash
ip route | grep 192.168
```

를 본다.

그리고:

```bash
ip route get $SERVER_IP
```

이게 굉장히 중요하다.

이 명령은 사실상 네가 계속 공부했던 질문:

> **"이 Pod IP로 가는 packet을 Linux kernel은 어디로 보내는가?"**

를 직접 묻는 것이다.

즉 Week 4에서 이야기한

```text
Destination Pod IP
       ↓
Routing decision
       ↓
Next hop / interface
```

를 실제 kernel에게 물어보는 것.

Calico 공식 troubleshooting 문서도 node routing table을 `ip route`로 확인하고, BGP로 학습된 route도 routing table에서 확인하는 방식을 사용한다. 

---

# 7. VXLAN interface를 직접 확인

Node에서:

```bash
ip -d link show vxlan.calico
```

여기서 VXLAN의 상세 속성을 볼 수 있다.

중요한 포인트는:

```text
vxlan.calico
   ↓
VXLAN
   ↓
UDP 4789
   ↓
VNI 4096
```

이다.

Calico 문서 기준 VXLAN 기본 VNI는 4096이다. 

---

# 8. 이제 tcpdump가 핵심

Node container 안에 `tcpdump`가 없으면:

```bash
apk add --no-cache tcpdump
```

그리고 **첫 번째 터미널**에서:

```bash
tcpdump -nn -i eth0 'udp port 4789'
```

이 상태로 기다린다.

다른 터미널에서:

```bash
kubectl exec calico-client -- curl -s http://$SERVER_IP >/dev/null
```

몇 번 실행한다.

그러면 이런 느낌의 packet을 볼 수 있다.

```text
Node2_IP → Node1_IP.4789
UDP
VXLAN
VNI 4096
```

그리고 VXLAN 안쪽에는 다시:

```text
Pod_Client_IP → Pod_Server_IP
TCP/80
```

가 들어간다.

즉 네가 Week 4에서 그렸던 그림을 **실제로 tcpdump로 확인하는 것**이다.

```text
Outer Packet

Node2 IP
    ↓
Node1 IP
    ↓
UDP 4789
    ↓
VXLAN
    ↓
VNI 4096
    ↓
Inner Packet
    ↓
Pod Client IP
    ↓
Pod Server IP
    ↓
TCP 80
```

실제 Calico VXLAN tcpdump에서도 outer Node IP와 UDP 4789, VXLAN header, 그리고 내부 Pod IP packet이 함께 관찰되는 형태다.

---

# 9. Inner packet만 보고 싶으면

다른 터미널에서:

```bash
tcpdump -nn -i vxlan.calico host $SERVER_IP
```

그러면 tunnel interface 기준으로:

```text
Pod Client IP → Pod Server IP
```

라는 **inner packet 관점**을 볼 수 있다.

즉 이렇게 비교하면 정말 좋다.

### `eth0`

```text
Node A → Node B
UDP 4789
VXLAN
  └── Pod A → Pod B
```

### `vxlan.calico`

```text
Pod A → Pod B
```

이걸 직접 보면 **Overlay라는 개념이 갑자기 굉장히 명확해진다.**

---

# 10. FDB까지 본다

같은 Node에서:

```bash
bridge fdb show dev vxlan.calico
```

이걸 보면 VXLAN forwarding에 필요한 FDB 정보를 확인할 수 있다.

여기서 질문은:

> "Pod IP를 가진 remote node로 가려면 VXLAN packet을 어느 VTEP로 보내야 하지?"

가 된다.

즉:

```text
Pod IP
 ↓
Route
 ↓
vxlan.calico
 ↓
VTEP 정보 / FDB
 ↓
Remote Node IP
```

라는 관계를 실제로 확인하는 것이다.

---

# 11. 그럼 Whisker는 어디에 쓰냐?

이제 이 상태에서:

```bash
kubectl get pods -n calico-system | grep -E 'goldmane|whisker'
```

를 실행한다.

현재 Calico 3.32 계열의 operator 설치에서는 Goldmane과 Whisker가 새 설치에 포함되어 있다. 

그리고:

```bash
kubectl port-forward -n calico-system service/whisker 8081:8081
```

브라우저:

```text
http://localhost:8081
```

그러면 flow를 볼 수 있다. Calico 문서에서도 이 방식으로 Whisker console을 port-forward해서 flow log를 확인하도록 안내한다. 

여기서 대략:

```text
calico-client
     ↓
calico-server

TCP
80
ALLOW
packet count
byte count
policy information
```

같은 **flow-level 정보**를 보는 거다.

---

# 12. 결국 각각 역할이 이렇게 나뉜다

이걸 외워두면 좋아.

| 도구                              | 보여주는 것                                 |
| ------------------------------- | -------------------------------------- |
| `Whisker`                       | 누가 누구와 통신했는가                           |
| `Goldmane`                      | flow aggregation / 정책 / packet·byte 통계 |
| `ip route`                      | kernel이 어느 경로로 보내는가                    |
| `ip route get`                  | 특정 destination에 대한 실제 routing decision |
| `ip -d link show vxlan.calico`  | VXLAN device 설정                        |
| `bridge fdb`                    | L2/VXLAN forwarding 정보                 |
| `ip neigh`                      | neighbor 정보                            |
| `tcpdump -i vxlan.calico`       | inner packet                           |
| `tcpdump -i eth0 udp port 4789` | outer VXLAN packet                     |

즉:

> **Whisker = "무슨 통신이 있었나?"**
> **routing/FDB = "어디로 보냈나?"**
> **tcpdump = "실제로 어떤 packet이 나갔나?"**

이렇게 보면 된다.

---

# 13. 그리고 이 다음에 아주 좋은 실습

지금 실습은 **Calico VXLAN**이야.

그 다음에는 동일한 Pod pair를 두고:

```text
VXLAN Always
        ↓
VXLAN CrossSubnet
        ↓
Native Routing
```

으로 바꿔가면서 비교하면 된다.

특히 Native Routing에서는:

```text
Pod A
 ↓
veth
 ↓
Linux route
 ↓
Node A
 ↓
Node B
 ↓
Linux route
 ↓
Pod B
```

가 되고, **VXLAN UDP 4789가 사라지는 것**을 tcpdump로 확인할 수 있다.

그리고 동시에:

```bash
ip route
```

에서 remote Pod CIDR route가 어떻게 설치되어 있는지 보고,

```text
BGP
 ↓
Route
 ↓
RIB
 ↓
FIB
 ↓
Packet forwarding
```

관계를 확인하면 돼.

    Calico 공식 troubleshooting에서도 BGP learned route가 Linux routing table에 반영되는 것을 `ip route`로 확인하도록 하고 있다.

    ---

    ### 네 Week 4 실습을 이렇게 잡으면 가장 좋음

    ```text
    Lab 1
    Calico IPAM
    Pool → /26 Block → Pod IP

            ↓

    Lab 2
    Same-node Pod → Pod
    veth → routing → veth

            ↓

    Lab 3
    Cross-node VXLAN
    Pod
    ↓
    cali*
    ↓
    route
    ↓
    vxlan.calico
    ↓
    eth0
    ↓
    UDP 4789 / VXLAN
    ↓
    eth0
    ↓
    vxlan.calico
    ↓
    route
    ↓
    cali*
    ↓
    Pod

            ↓

    Lab 4
    Whisker
    "이 flow가 존재한다"

            ↓

    Lab 5
    Native Routing + BGP
    "VXLAN 없이 어떤 route로 갔는가?"

            ↓

    Lab 6
    NetworkPolicy
    "어느 지점에서 DROP 되었는가?"
    ```
</details>



Ref:
- https://www.cisco.com/c/en/us/td/docs/dcn/whitepapers/cisco-nx-os-calico-network-design.html
- https://www.ibm.com/docs/tr/cloud-private/3.1.2?topic=ins-calico
- https://ninad-desai.medium.com/kubernetes-networking-demystified-overlay-vs-underlay-explained-291d5c6fb4fa
- https://collabnix.github.io/kubelabs/ClusterNetworking101/