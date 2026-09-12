# W1. Kubernetes & Linux Networking Overview

## 1. 학습 목적

애플리케이션마다 주소랑 포트를 따로 쓰게 하려면 네트워크 공간을 나눠야 하고, 나누고 나면 다시 이어줘야 하고, 이은 다음에는 어디로 갈지 패스를 정해야 합니다. 통신을 막거나 열어주는 것도 필요하다. Kubernetes에서도 이루어지는 일이지만 이를 이해하기 위해 Linux를 거쳐가겠습니다. 배워가는 과정이므로 각 요소마다 만약 이 컴포넌트가 없다면 무엇이 동작하지 않는지 생각하며 정리해보았습니다.

### Kubernetes, kind, 실습 환경 구성

- **Kubernetes:** 컨테이너를 관리해주는 시스템. 어떤 네트워크 방식이든 Pod끼리 통신이 가능하다면 OK.
- **kind:** 컨테이너를 노드처럼 사용해서 로컬에 Kubernetes 클러스터를 만들어주는 도구.
- **이번 실습**은 macOS에서 Docker Desktop의 Linux(Ubuntu) VM 위에 kind 클러스터를 노드 하나로 띄웠습니다. CNI 설정은 kindnet의 `ptp + host-local + portmap`입니다.

아래 정리한 내용은 위의 실습 환경에서 구성한 내용이 포함되어 있음을 유념해주세요.



## 2. 계층을 구분해서 이해하기


| 구분            | 대상                                    | 역할                     |
| ------------- | ------------------------------------- | ---------------------- |
| 커널과 네트워크 장치   | netns, veth, bridge, route, Netfilter | 패킷을 실제로 나누고 전달         |
| 사용자 공간의 도구    | containerd, CNI 실행 파일, ip, iptables   | 위의 설정을 만들고 수정          |
| Kubernetes 객체 | Pod, Service, EndpointSlice           | Kubernetes에서 다루는 단위    |
| 주소와 정책        | IPAM, CIDR, NetworkPolicy             | 주소를 나눠주고 통신 open/close |


### 용어 정리


| 용어                       | 정의                  | 역할                        |
| ------------------------ | ------------------- | ------------------------- |
| Network Namespace(netns) | 네트워크 공간을 나누는 커널 기능  | 인터페이스, 라우팅 테이블, 포트를 따로 가짐 |
| veth pair                | 쌍으로 만들어지는 가상 LAN카드  | 한쪽에 들어간 프레임이 반대쪽으로 나옴     |
| Linux bridge             | 커널 안의 가상 스위치        | MAC을 보고 프레임을 전달           |
| routing                  | IP 패킷의 경로를 정하는 것    | 나갈 인터페이스와 Next hop 선택     |
| ARP                      | IPv4에서 MAC을 찾는 프로토콜 | 같은 네트워크에 있는 상대의 MAC 확인    |
| TCP/IP                   | 통신 규칙 묶음이랑 그 구현     | 계층별로 통신 규칙 규정             |
| Netfilter                | 커널의 패킷 처리 기능        | 패킷을 걸러내거나 주소를 변환          |
| iptables                 | 규칙을 다루는 명령어         | 필터링이나 NAT 규칙을 보고 설정       |


[참고자료: Linux netns 설명](https://www.man7.org/linux/man-pages/man7/network_namespaces.7.html), [Netfilter 프로젝트](https://www.nftables.org/)



## 3. Kubernetes 컴포넌트의 역할


| 컴포넌트                    | 핵심 역할                        |
| ----------------------- | ---------------------------- |
| kube-apiserver          | API 요청을 받고 객체 상태를 알려줌        |
| etcd                    | 클러스터의 API 데이터를 저장            |
| kube-scheduler          | 아직 노드가 안 정해진 Pod의 노드를 고름     |
| kube-controller-manager | 원하는 상태랑 실제 상태를 맞춤            |
| kubelet                 | 자기 노드에 배정된 Pod를 띄우라고 런타임에 시킴 |
| containerd 등 런타임        | Pod sandbox랑 컨테이너를 실제로 띄움    |
| CNI 플러그인                | 네트워크를 연결                     |
| kube-proxy              | Service로 온 트래픽이 갈 길을 만들어줌    |


kubelet은 control-plane 노드에서도 동작. 각 컴포넌트는 API Server에 있는 상태를 보고 역할 수행.

 참고자료: [Kubernetes 네트워크 문서](https://kubernetes.io/docs/concepts/services-networking/)



## 4. TCP/IP


| 계층     | 예시         | 확인할 정보                |
| ------ | ---------- | --------------------- |
| 애플리케이션 | HTTP, DNS  | 요청 내용, 이름 해석          |
| 전송     | TCP, UDP   | 출발지/목적지 포트, 연결 상태     |
| 인터넷    | IPv4, IPv6 | 출발지/목적지 IP, 경로        |
| 링크     | Ethernet   | Next hop의 MAC, 프레임 전달 |


## 5. 격리와 연결은 서로 다른 문제

### 5.1 Network Namespace -&gt; 네트워크 공간 분리

커널은 하나 그대로 쓰면서 인터페이스, 경로, 포트 등을 공간별로 구분. Pod 하나에 컨테이너가 여러 개 있으면 그 컨테이너들은 네트워크 공간을 공유.

netns가 없으면 프로세스들이 주소랑 포트를 같이 쓰게 돼서 Pod마다 같은 포트를 따로 쓰게 해줄 수가 없음. 커널을 새로 띄우지 않고 공간만 나누는 거라 VM보다 가벼움. 대신 VM만큼 격리되지는 않음.

`hostNetwork: true`로 만든 Pod는 자기 공간을 안 쓰고 노드 네트워크를 그대로 사용.

### 5.2 veth pair -&gt; 두 공간 사이의 연결

양쪽 끝을 서로 다른 netns에 하나씩 두고 랜선처럼 쓰는 장치. 한쪽에 넣은 프레임이 반대쪽으로 나옴.

veth가 없으면 이 방식으로 만들어둔 연결이 없어지고 한쪽을 지우면 반대쪽도 같이 사라짐. 다시 만들 때는 이름만 맞추면 안 되고 주소랑 경로도 다시 넣어줘야 함. 참고자료: [veth 매뉴얼](https://man7.org/linux/man-pages/man4/veth.4.html)

### 5.3 Linux bridge -&gt; 여러 연결을 L2로 묶기

가상 인터페이스 여러 개를 스위치 하나에 꽂은 것처럼 묶어줌. bridge가 없으면 여기 꽂힌 포트끼리 프레임을 주고받지 못함. 다른 네트워크로 나가려면 bridge만으로는 안 되고 라우팅이 따로 있어야 함.

 참고자료: [CNI ptp 플러그인](https://www.cni.dev/plugins/current/main/ptp/)

## 6. Routing &amp; ARP: 방향을 고르고 Next hop 탐색

### Routing

목적지에 맞는 경로가 없으면 그 패킷은 못 나감. default route가 없어도 직접 연결된 경로가 있으면 같은 서브넷끼리는 통함.

경로는 보통 주소 하나가 아니라 IP prefix 단위. Pod마다 경로를 하나씩 넣을 수도 있는데 그러면 경로가 많아짐.

라우팅은 길을 고르는 거고 고른 길로 실제로 넘기는 건 forwarding.

### ARP

다음 홉의 MAC을 모르면 프레임을 못 보냄. 그래서 같은 네트워크에 특정 IP 쓰는 매체가 무엇인지 확인하고 답을 받아서 MAC을 알아냄. 한 번 알아낸 건 캐시에 저장해둠.

IPv6에서는 ARP 사용 X.

```text
목적지 IP 확인 → route 조회 → 출력 인터페이스/다음 홉 결정
                                ↓
                 필요한 경우 다음 홉의 MAC 확인
                                ↓
                         Ethernet 프레임 전송
```



## 7. Netfilter &amp; iptables: 경로 중간에서 처리

Netfilter는 커널이 패킷을 처리하는 중간중간에 규칙을 걸 수 있게 해주는 기능. iptables는 그 규칙을 사람이 넣고 빼는 명령어.

패킷이 지나가는 자리는 대략 이런 구성.

```text
수신 → PREROUTING → 경로 판단 ┬→ INPUT → 로컬 프로세스
                           └→ FORWARD → POSTROUTING → 송신

로컬 프로세스 → 경로 선택 → OUTPUT → POSTROUTING → 송신
```

규칙은 위에서부터 순서대로 보기 때문에 순서랑 어느 자리에 넣었는지가 중요. 그리고 규칙을 지운다고 항상 통신이 끊기는 것도 아님. DROP 규칙을 지우면 오히려 다시 통함.



## 8. 실습

### 실습 환경의 경계

```text
macOS
└─ Docker Desktop의 Linux VM
   └─ kind 노드 컨테이너: cni-week1-control-plane
      ├─ 실제 Kubernetes Pod: kindnet의 ptp 연결
      └─ 별도 Linux 실험: study-w1-a / study-w1-sw / study-w1-b
```

따로 만든 Linux 실습은 아래처럼 구성.

```text
A netns                 SW netns                    B netns
192.0.2.1/24                                      192.0.2.2/24
eth0 ← veth pair → a-port — br-study — b-port ← veth pair → eth0
```

예측 → 조작 → 관찰 → 복구 순서로 실습 진행.


| 실습                | 실행 전 예측                                  | 확인                         |
| ----------------- | ---------------------------------------- | -------------------------- |
| bridge 포트 분리      | B 쪽 포트를 떼면 A → B ping 실패, 다시 붙이면 회복      | 포트 연결 상태와 ping 결과          |
| ICMP DROP 추가 및 제거 | B INPUT에서 echo request를 버리면 실패, 규칙 빼면 회복 | DROP 카운터와 ping 결과          |
| veth 한쪽 삭제        | peer랑 주소, 경로까지 같이 사라져서 실패                | B의 인터페이스와 경로를 다시 만든 뒤 ping |


포트를 떼거나 DROP 규칙을 넣었을 때: ping 실패, 복구하니 다시 회복함(예측대로 진행)

veth 한쪽을 삭제했을 떄: B의 eth0랑 연결 경로도 같이 없어짐. veth를 다시 만들었는데도 ping이 안 감. B의 MAC은 새로 바뀌었는데 A에는 예전 MAC이 남아 있었고 상태가 STALE → DELAY → FAILED로 변화. A에서 그 항목을 지운 후 다시 ping을 쏜 뒤에야 새 MAC을 찾아왔고 3번 다 성공.



## 9. 질문

1. netns는 있는데 veth가 없으면 무조건 통신이 안 되나? 다른 방법도 있나?
2. ping이 안 될 때 ARP 문제인지 규칙 문제인지 어떻게 구분하나?
3. 인터페이스를 지우는 거랑 link down 시키는 건 복구할 때 뭐가 다른가?