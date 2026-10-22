# W8. Kubernetes Networking Common Issue Troubleshooting & Operations

## 1. 주제 선정: 통신이 실패한 위치 정확히 잡아내기

이번 주에는 CNI 트러블슈팅의 넓은 흐름을 따라가되, 동작 중인 클러스터에서 재현할 수 있는 문제를 직접 확인해봤습니다. Pod IP는 통하지만 Service IP가 실패하는 경우와, egress NetworkPolicy가 실제 패킷을 막는 경우를 비교했습니다.

## 2. 통신이 멈춘 단계를 구분

Pod 생성, 노드 배정, IP 할당, 목적지별 통신 순으로 나눠 봄. 노드에 배정되지 않은 Pod는 CNI 단계 전이고, Service IP만 실패하면 Service와 백엔드 연결부터 비교.

| 관찰 상태                        | 먼저 확인한 항목                                 | 이번 실습에서 본 의미                  |
| ---------------------------- | ----------------------------------------- | ----------------------------- |
| Pod 생성이 `Forbidden`으로 거부     | API 응답, ResourceQuota                     | API 서버가 Pod 객체 생성 전에 거부       |
| Pod가 `Pending`, Node가 비어 있음  | Events, selector, taint, requests         | 스케줄러가 배치할 노드를 찾지 못함           |
| 노드는 배정됐지만 IP가 없거나 sandbox 실패 | Pod Events, kubelet/runtime, CNI와 IPAM 로그 | Pod 네트워크 준비 단계에서 실패했을 가능성이 있음 |
| Pod IP 요청부터 실패               | 정책, 노드 간 경로, CNI 상태                       | Pod 간 데이터 경로를 조사함             |
| Pod IP는 통하지만 Service IP 실패   | Service, EndpointSlice, 포트, kube-proxy    | Service 경로의 설정과 전달을 조사함       |

참고자료: [Service 디버깅](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/), [ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/)

## 3. 실습 환경과 정상 기준선

실습 클러스터는 control-plane과 worker 한 대씩 사용.

| 항목 | 실제 구성 |
| --- | --- |
| Kubernetes / Cilium | 1.35.8 / 1.20.2 |
| Cilium IPAM / routing | Kubernetes IPAM / tunnel |
| 터널 / kube-proxy | VXLAN / kube-proxy 유지 |
| client | worker |
| server | worker |
| client-cross-node | control-plane |
| server Service | ClusterIP port 80, selector `app=server` |

같은 노드 Pod IP, Service IP, control-plane에서 worker Pod IP 요청은 모두 200이었음. 이 경로의 기준선으로 장애 상황을 추정.

이번 구성은 kube-proxy를 사용하므로 Service 실습으로 kube-proxy 장애 자체를 검증한 것은 아님. DNS 장애나 CNI sandbox 실패도 만들지 않았음.

## 4. Pod IP는 성공하고 Service IP는 실패한 경우

selector를 `app=missing`으로 바꾸자 EndpointSlice에서 서버 Pod가 빠졌고 Pod IP는 200, Service IP는 시간 초과였음. `targetPort`를 8080으로 바꾸자 EndpointSlice는 8080을 가리켰지만 nginx는 80에서 수신 중이었음.

| 변경한 값 | Pod IP 요청 | Service IP 요청 | 확인한 원인 |
| --- | --- | --- | --- |
| selector `app=missing` | HTTP 200 | 시간 초과, curl 종료 코드 28 | EndpointSlice에 백엔드가 없음 |
| `targetPort: 8080` | HTTP 200 | 연결 거부, curl 종료 코드 7 | 서버가 8080에서 수신하지 않음 |

각 설정을 복구하자 Service 요청도 성공함. 두 경우 모두 Service 설정 오류였음. Pod IP와 Service IP를 비교하고 EndpointSlice 주소와 포트를 대조해 원인을 좁힘. 

참고자료: [Kubernetes Service 디버깅](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/), [curl 종료 코드](https://curl.se/libcurl/c/libcurl-errors.html)

## 5. Hubble Flow에서 egress 정책 차단 확인

`client-egress-lab` 정책이 `app=client` Pod의 egress를 허용하지 않자 Service와 서버 Pod IP 요청이 모두 실패함. Hubble에서 client가 server TCP 80으로 보낸 SYN이 `EGRESS`, `DROPPED`, `POLICY_DENIED`로 기록됨.

| 관측한 Pod / Flow | 실제 결과 | 판단 |
| --- | --- | --- |
| worker의 `app=client` | Pod IP와 Service IP 요청 실패 | 정책 선택 대상의 egress가 차단됨 |
| Hubble: client Pod → server Pod, TCP 80 SYN | `EGRESS`, `DROPPED`, `POLICY_DENIED` | 정책 차단을 보여주는 근거임 |
| control-plane의 `client-cross-node` | worker Pod IP 요청 성공 | `app=client` 선택자에 포함되지 않음 |

client에서 server TCP 80으로 향하는 egress를 허용하자 Service 요청은 성공했지만 DNS와 외부 통신은 실패함. egress 전체 허용 후 내부 Service와 외부 HTTPS 요청이 성공함. 실습 후에도 이 정책 객체는 남아 있고 모든 egress를 허용함.

`DROPPED`와 `POLICY_DENIED`로 정책 차단을 특정함. `FORWARDED`는 해당 관측 지점에서 다음 처리 대상으로 전달했다는 뜻임. 최종 성공 여부는 응답과 목적지 쪽 기록까지 확인해야 함.

참고자료: [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/), [Cilium 트러블슈팅](https://docs.cilium.io/en/stable/operations/troubleshooting/)

## 6. IPAM 주소와 Pod 용량 확인

각 노드의 Pod CIDR은 `/24`, Pod 수용 한도는 110개였음. Cilium IPAM은 control-plane `4/254`, worker `10/254`로 주소 여유를 보였음. 실제 IP 고갈이나 `FailedCreatePodSandBox`는 재현하지 않음.

주소가 남아도 Pod 수 한도, CPU와 메모리 요청량, taint나 selector가 먼저 병목이 될 수 있음. 노드 배정 뒤 sandbox나 IP 할당이 실패하면 Pod 이벤트, 노드별 CIDR과 한도, Cilium IPAM 상태를 함께 봄.

```bash
kubectl describe pod -n <namespace> <pod>
kubectl get events -A --sort-by=.lastTimestamp
kubectl get nodes -o custom-columns=NAME:.metadata.name,PODCIDR:.spec.podCIDR,PODS:.status.allocatable.pods
kubectl -n kube-system exec <cilium-pod> -- cilium-dbg status --verbose
```

실습 환경에서 Kubernetes IPAM은 AWS VPC CNI의 ENI와 서브넷 주소 할당 구조와 달라서 일반화는 불가능. OLX Autos의 EKS 사례에서는 Pod 생성이 급증해 미리 준비한 IP를 소진하면서 sandbox 생성이 실패함. 서브넷 잔여 IP와 노드가 바로 쓸 수 있는 IP를 구분해야 함.

참고자료: [Cilium IPAM 상태와 진단](https://docs.cilium.io/en/stable/operations/troubleshooting/), [OLX Autos의 EKS IP 확장 사례](https://aws.amazon.com/blogs/containers/scale-from-100-to-10000-pods-on-amazon-eks/)

## 7. VXLAN, MTU, 노드 방화벽

Cilium 설정은 `routing-mode=tunnel`, `tunnel-protocol=vxlan`이었고 MTU는 직접 지정되지 않았음. `packetization-layer-pmtud-mode`는 `blackhole`이었음. 두 agent와 client Pod에서 확인한 값은 다음과 같음.

| 확인 위치 | 값 |
| --- | ---: |
| Cilium `DeviceMTU` | 65520 |
| Cilium `RouteMTU` | 65470 |
| Cilium `RoutePostEncryptMTU` | 65520 |
| client Pod `eth0` MTU | 65520 |

Pod `eth0`의 65520과 `RouteMTU` 65470이 다른 것은 오류가 아님. `DeviceMTU`는 Pod 쪽 장치에, `RouteMTU`는 터널 오버헤드를 반영한 경로에 쓰임. VXLAN over IPv4 오버헤드는 50바이트임. kind/Docker의 큰 값은 Linux route MTU 상한에 따른 것으로, 물리 Ethernet MTU를 뜻하지 않음.

underlay MTU가 1500이면 VXLAN 경로의 Pod MTU는 보통 1450임. 경로 MTU가 맞으면 일반 TCP는 동작함. MTU 불일치로 큰 패킷만 유실될 수 있지만 이번에는 재현하지 않았음.

다른 노드 통신만 실패하면 터널 경로와 방화벽도 확인함. VXLAN은 노드 간 UDP 8472가 필요함. Pod egress 허용과 별개로 노드 방화벽이 이 포트를 막을 수 있음. cross-node 요청은 성공했지만 방화벽 차단은 시험하지 않았음.

참고자료: [Cilium MTU 계산 코드 v1.20.2](https://github.com/cilium/cilium/blob/v1.20.2/pkg/mtu/mtu.go), [Cilium 라우팅](https://docs.cilium.io/en/stable/network/concepts/routing/), [Cilium 방화벽 요구사항](https://docs.cilium.io/en/stable/operations/system_requirements/)

## 8. 질문

1. 작은 cross-node HTTP 요청은 성공하지만 큰 응답만 멈춘다면, MTU 문제와 정책 또는 방화벽 문제를 구분하는 실험을 어떻게 설계할 수 있을지?

2. Pod가 급증하는 환경에서 IP 여유가 줄어들 때, 서브넷 주소, 노드별 Pod CIDR, 노드의 Pod 상한, CPU와 메모리 중 무엇을 먼저 확장해야 할까? 고갈 전에 어떤 지표와 경보를 두어야 불필요한 확장을 피할 수 있을지?