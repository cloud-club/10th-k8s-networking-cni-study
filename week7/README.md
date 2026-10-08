# Week 7. eBPF Deep Dive & Hubble

eBPF의 내부 동작과 Cilium의 활용 방식을 조금 더 깊게 이해하고, Hubble을 통해 실제 네트워크 흐름을 분석합니다. week5에서 학습한 내용을 토대로 더 알아보고 싶은 부분을 학습하면 됩니다.

## 학습 포인트

* **eBPF 동작 원리**

  * Program / Verifier / JIT / Hook
  * eBPF Program이 Kernel에서 실행되는 과정

* **BPF Map & Cilium Datapath**

  * BPF Map의 역할
  * `ipcache`, LB / Policy Map
  * eBPF Program이 Map을 참조해 Packet을 처리하는 과정

* **Hubble & Troubleshooting**

  * eBPF Event → Hubble Flow
  * `FORWARDED / DROPPED` Flow 분석
  * Hubble을 활용한 NetworkPolicy / 통신 문제 추적

## 정리할 내용

* eBPF Program이 **어떻게 Kernel에서 실행되는가**
* Cilium이 **BPF Map을 이용해 네트워크 상태를 관리하는 방식**
* eBPF Datapath에서 발생한 이벤트가 **Hubble Flow로 연결되는 과정**
* Hubble을 이용해 **통신 성공/실패 원인을 추적하는 방법**

## Hands-on — 선택 사항

* `bpftool`로 eBPF Program / Map 확인
* Cilium BPF Map 및 Hubble Flow 확인
* NetworkPolicy 적용 후 `FORWARDED / DROPPED` Flow 비교
