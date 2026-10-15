# Week 7. eBPF Deep Dive: Kernel에서 Program이 실행되는 원리

## 1. 이번 주의 핵심 질문

Week 5에서는 Cilium이 eBPF를 이용해 Network, Service, Policy와 Observability를 구현한다는 전체 구조를 살펴봤다.

```text
eBPF Source
  → Bytecode
  → Kernel에 Load
  → Verifier의 정적 검증
  → JIT 또는 Interpreter
  → Hook에 Attach
  → Event가 발생할 때 실행
  → Map을 통해 상태 공유
```

핵심 질문은 다음과 같다.

1. eBPF Program은 일반 User Space Program과 무엇이 다른가?
2. eBPF Virtual Machine은 어떤 Register와 Calling Convention을 사용하는가?
3. Verifier는 Program이 안전하다는 것을 어떻게 판단하는가?
4. Program Type, Context, Helper와 Hook은 어떤 관계인가?
5. Program이 종료된 뒤에도 상태를 유지할 수 있는 이유는 무엇인가?
6. Tail Call과 JIT는 실행 구조에서 어떤 역할을 하는가?

> 이 문서에서 Cilium과 Hubble은 eBPF를 이해하기 위한 중심 주제가 아니라, 마지막에 살펴보는 실제 활용 사례로만 다룬다.

### 1.1 먼저 알아보는 eBPF

eBPF는 Linux Kernel 안의 정해진 지점에서, Kernel을 직접 수정하지 않고도 사용자가 만든 작은 Program을 실행하게 해 주는 기술이다. 여기서 Program은 컴퓨터가 수행할 명령을 순서대로 작성한 것으로, eBPF Program은 일반 Application보다 할 수 있는 일이 제한되어 있는 대신 Kernel Event가 발생한 바로 그 위치에서 동작할 수 있다.

```text
Network Packet 도착, System Call 실행 같은 Event
  → Kernel의 Hook에서 eBPF Program 호출
  → Packet 검사, 통계 집계, 접근 제어 같은 짧은 작업 수행
  → 결과를 반환하고 이번 실행 종료
```

Kernel은 eBPF Program을 실행하기 전에 Verifier로 안전성을 검사한다. Program은 허용된 Context와 Helper만 사용하며, 여러 호출에 걸쳐 보관할 상태는 BPF Map에 저장한다. 따라서 eBPF의 핵심은 다음 세 문장으로 요약할 수 있다.

```text
어디서 실행할지       → Hook
안전하게 실행 가능한지 → Verifier
실행 사이의 상태       → BPF Map
```

---

## 2. 왜 Kernel에서 Program을 실행하려는가

### 2.1 User Space에서 처리하는 방법

Network Packet이나 System Event를 User Space Program에서 처리하려면 Kernel이 해당 정보를 User Space로 전달해야 한다.

```text
Kernel Event
  → User Space로 전달
  → User Program이 처리
  → 필요한 경우 다시 System Call
  → Kernel에 결과 반영
```

User Space는 격리와 개발 편의성이 좋지만, 매우 자주 발생하는 Event를 처리할 때는 Context Switch, Data 전달과 System Call 비용을 고려해야 한다.

Kernel Module은 실행 중인 Kernel에 기능을 추가하는 Code 묶음이다. Device Driver나 File System처럼 Kernel 권한이 필요한 기능을 구현할 수 있지만, 일반적인 Module Code에는 eBPF Verifier와 같은 실행 전 안전성 검사가 적용되지 않는다. 잘못된 Pointer 접근이나 동기화 오류가 있으면 해당 Process만이 아니라 Kernel 전체의 장애로 이어질 수 있고, Kernel 내부 API 변화에 따른 Version 호환도 관리해야 한다.

필요한 것은 다음 두 조건을 함께 만족하는 실행 방법이다.

```text
Kernel의 Event 지점에서 바로 실행
                    +
임의의 Kernel Code보다 제한되고 검증된 실행
```

eBPF는 Kernel이 제공하는 Hook에 Program을 연결하고, Event가 발생하면 검증된 Program을 Kernel Context에서 실행한다.

### 2.2 eBPF는 Event-driven Program이다

일반 Application은 Process로 실행되어 Scheduler의 관리 아래 계속 살아 있을 수 있다. eBPF Program은 특정 Hook의 Event가 발생할 때 호출되고, 짧은 작업을 수행한 뒤 반환한다.

```text
일반 Application

Process 시작
  → 자신의 Address Space 사용
  → System Call로 Kernel 기능 요청
  → 반복 실행
  → Process 종료
```

```text
eBPF Program

Kernel Event 발생
  → Hook에 연결된 Program 호출
  → Context 검사
  → Helper와 Map 사용
  → Action 또는 값을 반환
  → 이번 호출 종료
```


| 구분        | 일반 User Space Program          | eBPF Program                                |
| --------- | ------------------------------ | ------------------------------------------- |
| 실행 위치     | User Space                     | Kernel Context                              |
| 실행 시점     | Process가 Scheduling될 때         | 연결된 Hook의 Event가 발생할 때                      |
| 입력        | Argument, File, Socket 등       | Program Type별 Context                       |
| Memory    | Process의 Virtual Address Space | Verifier가 허용한 Register, Stack, Context와 Map |
| Kernel 기능 | System Call                    | BPF Helper 또는 허용된 Kernel Function           |
| 상태 유지     | Heap, File, Database 등         | BPF Map                                     |
| 결과        | Application Logic              | Hook별 반환값과 Side Effect                      |


따라서 eBPF는 “Kernel에서 계속 실행되는 Daemon”이 아니다. Kernel Event에 반응하는 Callback Program에 가깝다.

---



## 3. 하나의 eBPF Application은 두 부분으로 구성된다

eBPF Program만 작성해서는 아직 실행할 수 없다. 누군가 Program을 Kernel에 넣고, 실행할 Hook에 연결하고, 설정을 전달하고, 결과를 읽어야 한다. 그래서 실제 eBPF Application은 보통 User Space에서 관리하는 부분과 Kernel에서 Event를 처리하는 부분으로 나뉜다.

예를 들어 “목적지 Port가 8080인 Packet 수를 센다”는 Application을 생각해 보자.

```text
User Space Loader / Controller
  → eBPF Program과 Counter Map을 Kernel에 Load
  → Program을 Network Interface의 XDP Hook에 Attach
  → 일정 시간마다 Counter Map을 읽어 화면에 출력

Kernel Space의 eBPF Program
  → Packet이 들어올 때마다 호출
  → 목적지 Port가 8080이면 Counter Map의 값을 증가
  → Packet을 계속 통과시킴
```

빠른 Packet 처리는 Kernel에서 하지만, 설치·설정·결과 표시는 User Space가 담당한다. 이 역할 분담을 알아야 “누가 Program을 Load하는가?”, “Program이 끝났는데 Counter는 어디에 남는가?” 같은 질문에 답할 수 있다.

```text
User Space                              Kernel Space

Loader / Controller
  ├─ Object File Open
  ├─ Program과 Map Load  ─────────────► eBPF Program
  ├─ Hook에 Attach       ─────────────► Hook
  ├─ Map 초기화·조회     ◄────────────► BPF Map
  └─ Event 읽기           ◄──────────── Event Buffer
```


| 구성                | 역할                                         |
| ----------------- | ------------------------------------------ |
| User Space Loader | Object를 열고 Program과 Map을 생성하며 Kernel에 Load |
| Controller        | Map을 갱신하고 Event를 읽으며 Program의 생명주기 관리      |
| eBPF Program      | Hook의 Event가 발생했을 때 실제 Logic 실행            |
| BPF Map           | 호출 사이의 상태와 User/Kernel 공유 Data 저장          |


`libbpf`, Go의 `cilium/ebpf`, Rust의 Aya 같은 Library는 `bpf()` System Call을 직접 다루는 대신 Object 열기, Program·Map 생성, Attach와 정리 같은 User Space 작업을 더 안전하고 편리하게 처리해 준다.

아래의 Open → Load → Attach → Execute → Detach는 Packet 한 개가 처리되는 주기가 아니라 eBPF Application을 준비하고 운영한 뒤 종료하는 생명주기다.

| 단계 | 주체와 주로 사용하는 수단 | 하는 일 |
| --- | --- | --- |
| Open | Loader, `libbpf` | ELF Object를 읽고 Program·Map·Section을 발견한다. |
| Load | Loader, `bpf()` System Call | Map을 만들고 Program을 Kernel에 전달해 Verifier 검사를 받는다. |
| Attach | Loader, BPF Link 또는 Hook별 API | Load된 Program을 실제 Kernel Hook에 연결한다. |
| Execute | Kernel | Hook의 Event가 발생할 때 Program을 호출한다. |
| Detach·정리 | Controller, Link와 FD 관리 | 연결과 참조를 제거하고 더 이상 쓰지 않는 Kernel Object를 정리한다. |

ELF(Executable and Linkable Format)는 Compiler가 Code와 Data를 Section 단위로 담는 Object File 형식이다. eBPF에서는 실행할 Instruction뿐 아니라 Map 정의, License, BTF Type 정보와 `SEC("xdp")`, `SEC("tracepoint/...")` 같은 Section 이름도 담을 수 있다. Loader는 ELF를 읽어 어떤 Program과 Map이 있으며 각 Program을 어떤 종류로 해석할지 알아낸다.

Hook은 Kernel 처리 경로 중 eBPF Program이 호출될 수 있는 연결 지점이다. “Program이 무엇을 하는가”와 별개로 “언제, 어떤 Context와 함께 실행되는가”를 정한다.

| 대표 Hook | Event가 발생하는 위치 | 대표 용도 |
| --- | --- | --- |
| XDP | Network Driver의 매우 이른 Packet 수신 경로 | 빠른 Drop, Redirect, Load Balancing |
| TC ingress/egress | Network Stack의 입·출력 Traffic Control 경로 | Policy, Packet 수정, Traffic 관측 |
| Tracepoint | Kernel이 미리 정의한 Trace Event | System Call, Scheduler, Network Event 추적 |
| Kprobe 계열 | 선택한 Kernel Function의 진입·반환 지점 | 특정 Kernel 동작의 동적 추적 |
| cgroup Hook | cgroup에 속한 Process의 Socket·Packet 동작 | Workload 단위 Network 제어와 관측 |
| LSM Hook | File, Process, Socket 관련 보안 판단 지점 | 접근 제어와 Audit |

이제 `libbpf`를 기준으로 각 생명주기를 자세히 살펴본다.

### 3.1 Open

Loader가 Compiler가 만든 ELF Object를 읽고 Program, Map, Global Variable과 Section 정보를 찾는다. 아직 Kernel에 Program이 만들어진 것은 아니다.

### 3.2 Load

Loader가 Map을 만들고 Relocation을 처리한 뒤 `bpf()` System Call을 이용해 Program을 Kernel에 전달한다. 이때 Verifier가 Program을 검사한다.

```text
ELF Object
  → Map 생성
  → Relocation 처리
  → BPF_PROG_LOAD
  → Verifier
  → 성공하면 Kernel Program Object 생성
```



### 3.3 Attach

Load된 Program을 XDP, TC, Tracepoint, cgroup 같은 실제 Hook에 연결한다. Load 성공과 Attach 성공은 서로 다른 단계다.

```text
Load만 성공
  → Kernel에 Program은 존재
  → Event에서는 아직 실행되지 않을 수 있음

Attach까지 성공
  → Hook Event 발생
  → Program 실행
```



### 3.4 Execute

Hook의 Event가 발생하면 Kernel이 Program Type에 맞는 Context를 전달해 Program을 호출한다. Program은 Map을 읽거나 Helper를 호출하고 반환값을 Kernel에 돌려준다.

### 3.5 Detach와 정리

Link를 제거하면 Hook에서 Program 실행이 중단된다. 더 이상 Program이나 Map을 참조하는 File Descriptor와 Pin이 없다면 Kernel Object도 정리될 수 있다.

[Linux Kernel libbpf Overview](https://docs.kernel.org/bpf/libbpf/libbpf_overview.html)

---



## 4. eBPF Virtual Machine과 Instruction Set



### 4.1 Source Code가 바로 실행되는 것은 아니다

eBPF Program은 일반적으로 제한된 C로 작성하고 Clang과 LLVM을 이용해 eBPF Instruction으로 Compile한다.

- Clang은 C Source를 읽고 문법·Type을 검사한 뒤 LLVM이 처리할 중간 표현으로 바꾸는 Compiler Frontend다.
- LLVM은 중간 표현을 최적화하고 목표 Architecture에 맞는 명령을 만드는 Compiler 기반 기술이다.
- `-target bpf`를 지정하면 LLVM의 BPF Backend가 x86-64나 arm64 명령이 아니라 eBPF Virtual Machine용 Instruction을 만든다.

eBPF Instruction이 필요한 이유는 개발자의 C Source를 Kernel이 직접 실행하거나 검증하지 않기 때문이다. Kernel은 Architecture에 독립적인 eBPF Instruction을 공통 입력으로 받아 Verifier로 검사한 뒤, Interpreter로 해석하거나 현재 CPU의 Native Instruction으로 JIT Compile해 실행한다.

```text
C Source
  → clang/LLVM의 BPF Backend
  → eBPF Instruction을 포함한 ELF Object
  → Loader가 Kernel에 전달
  → Verifier가 Instruction 검사
  → Interpreter 또는 JIT로 실행
```

Kernel이 검증하는 대상은 C Source 자체가 아니라 Compiler가 생성한 eBPF Instruction이다. 같은 Source라도 Compiler Optimization과 Target 설정에 따라 Verifier가 보는 Instruction은 달라질 수 있다.

### 4.2 Register와 Calling Convention

eBPF는 64-bit Register 기반 Instruction Set을 사용한다. 10개의 General Purpose Register와 Read-only Frame Pointer가 있다.


| Register  | 역할                                     |
| --------- | -------------------------------------- |
| `R0`      | Function의 반환값, Program 종료 시 반환값        |
| `R1`~`R5` | Helper와 Function 호출의 Argument          |
| `R6`~`R9` | 호출 뒤에도 보존되는 Callee-saved Register      |
| `R10`     | Stack 접근에 사용하는 Read-only Frame Pointer |


`R0`~`R5`는 호출 과정에서 값이 바뀔 수 있는 Scratch Register다. 호출 이후에도 값이 필요하다면 Compiler가 `R6`~`R9` 또는 Stack을 사용하도록 Instruction을 만든다.

```text
Helper 호출 전

R1 = 첫 번째 Argument
R2 = 두 번째 Argument
R3 = 세 번째 Argument
R4 = 네 번째 Argument
R5 = 다섯 번째 Argument
call helper
R0 = Helper 반환값
```

예를 들어 Map에서 값을 찾은 뒤 원래 Context를 계속 사용해야 한다고 가정해 보자. Program 시작 시 `R1`에 있던 Context Pointer를 그대로 두고 Helper를 호출하면 `R1`~`R5`의 기존 값은 호출 뒤 사용할 수 없다. Compiler는 필요한 Context Pointer를 보존 Register인 `R6`로 옮길 수 있다.

```text
Program 시작
  R1 = Context Pointer

Helper 호출 준비
  R6 = R1              ← 호출 뒤에도 쓸 Context를 보관
  R1 = Map 참조        ← Helper의 첫 번째 Argument
  R2 = Key 주소        ← Helper의 두 번째 Argument
  call map_lookup

Helper 호출 뒤
  R0 = Map Value Pointer 또는 NULL
  R1~R5 = 이전 값 사용 불가
  R6 = Context Pointer ← 보존되어 계속 사용 가능
```

이것은 C 개발자가 보통 Register 번호를 직접 지정한다는 뜻이 아니다. Compiler가 Calling Convention에 맞춰 C Variable을 Register나 Stack에 배치하고, Verifier는 만들어진 Instruction에서 각 Register의 상태를 추적한다.

이 Calling Convention은 여러 64-bit CPU Architecture의 Register Calling Convention과 효율적으로 Mapping할 수 있도록 설계됐다. [Linux Kernel BPF Instruction Set](https://docs.kernel.org/bpf/standardization/instruction-set.html)

### 4.3 C Code와 Instruction의 관계

다음은 Map에서 Key를 찾는 개념적인 C Code다.

```c
__u32 key = 1;
__u64 *value;

value = bpf_map_lookup_elem(&counter_map, &key);
if (value)
    __sync_fetch_and_add(value, 1);
```

위 Code는 Key `1`에 해당하는 Counter를 Map에서 찾아, Entry가 존재할 때 1을 더한다. `&counter_map`은 조회할 Map, `&key`는 찾을 Key의 주소이며, Lookup 결과는 Entry가 없을 수 있으므로 Pointer를 사용하기 전에 `if (value)`로 NULL 여부를 확인한다. 여러 CPU가 같은 Counter를 갱신할 수 있어 예시에서는 Atomic Add를 의미하는 `__sync_fetch_and_add()`를 사용했다.

Instruction 관점에서는 다음과 같은 작업으로 나뉜다.

```text
1. Stack에 key 저장
2. R1에 Map 참조 준비
3. R2에 key의 Stack 주소 준비
4. Map Lookup Helper 호출
5. R0가 NULL인지 분기
6. NULL이 아니면 Map Value Pointer로 값 갱신
7. Hook에 맞는 값을 R0에 두고 종료
```

Verifier는 `R0`를 단순한 64-bit 숫자로만 보지 않는다. Helper 호출 결과라는 사실을 이용해 `NULL일 수도 있는 Map Value Pointer`로 추적한다. `if (value)` 검사를 통과한 경로에서만 안전한 Pointer로 사용할 수 있다.

---



## 5. Program Type, Hook, Context와 반환값



### 5.1 Program Type이 실행 계약을 정한다

Program Type은 “이 eBPF Program이 어떤 종류의 일을 하기 위해 실행되는가”를 Kernel에 알리는 분류다. Loader는 ELF Section이나 명시적인 설정을 바탕으로 Program Type을 정해 `BPF_PROG_LOAD`를 요청하고, Kernel은 그 Type에 맞는 검증 규칙과 실행 방식을 적용한다. 이는 단순한 이름표가 아니라 Program이 지켜야 할 Interface 계약이다.

eBPF Program은 모든 Kernel Event에서 같은 방식으로 실행되지 않는다. Program Type은 다음 계약을 결정한다.

```text
Program Type
  ├─ 어떤 Hook에 Attach할 수 있는가
  ├─ 어떤 Context를 입력으로 받는가
  ├─ 어떤 Helper를 사용할 수 있는가
  └─ 반환값을 Kernel이 어떻게 해석하는가
```


| 목적                 | 대표 Program Type                    | Context 예시                  | 반환값의 의미 예시                      |
| ------------------ | ---------------------------------- | --------------------------- | ------------------------------- |
| 이른 Packet 수신 처리    | XDP                                | `struct xdp_md`             | PASS, DROP, TX, REDIRECT        |
| Traffic Control    | `SCHED_CLS`, `SCHED_ACT`           | `struct __sk_buff`          | 통과, Drop, Redirect 등의 TC Action |
| Kernel Event 추적    | Tracepoint, Kprobe 계열              | Event/Function별 Context     | 관측 Logic에 따른 값                  |
| cgroup의 Network 제어 | `CGROUP_SKB`, `CGROUP_SOCK_ADDR` 등 | Packet 또는 Socket 주소 Context | 허용·거부, 주소 처리 등                  |
| Kernel 보안 Hook     | LSM                                | LSM Hook별 Context           | 허용 또는 Error Code                |




### 5.2 XDP와 TC 비교

```text
NIC Packet 수신
  → XDP Hook
  → skb 생성과 Network Stack 처리
  → TC ingress
  → Routing / Protocol 처리
  → Socket
```


| 구분        | XDP                               | TC                                    |
| --------- | --------------------------------- | ------------------------------------- |
| 실행 위치     | Driver의 매우 이른 수신 경로               | Generic Network Stack의 ingress/egress |
| Packet 표현 | `xdp_buff` 계열 Context             | `sk_buff` 기반 Context                  |
| Metadata  | 아직 적음                             | 더 많은 Network Metadata 사용 가능           |
| 방향        | 기본적으로 ingress                     | ingress와 egress                       |
| 대표 목적     | 빠른 Drop, Redirect, Load Balancing | Policy, Packet 수정, Redirect, 관측       |


XDP가 더 이른 지점에서 실행되므로 항상 더 좋은 것은 아니다. 아직 `skb`가 만들어지지 않아 사용할 수 있는 Metadata와 Kernel 기능이 적다. 필요한 Context와 기능에 맞춰 Hook을 선택해야 한다.

### 5.3 Context는 임의의 Kernel Pointer가 아니다

Program이 시작될 때 `R1`에는 Program Type에 맞는 Context Pointer가 들어온다. XDP라면 Packet의 시작과 끝을 나타내는 정보가 있고, Socket 계열이라면 주소와 Port 같은 정보가 있다.

Program은 Context의 모든 Field를 자유롭게 읽고 쓸 수 없다.

- Program Type에 공개된 Field만 접근 가능
- Field마다 Read-only 또는 Read-write 여부가 다를 수 있음
- Packet Data를 읽기 전에 Boundary Check 필요
- Helper 호출 뒤 Pointer의 유효성이 달라질 수 있음
- Kernel Version에 따라 사용할 수 있는 Field와 기능이 다를 수 있음

따라서 같은 C Code라도 Program Type이 다르면 Compile 또는 Load 과정에서 사용할 수 없는 Context와 Helper 때문에 실패할 수 있다.

예를 들어 XDP Program은 `struct xdp_md` Context에서 `data`와 `data_end`를 읽어 Packet 범위를 검사하고 `XDP_PASS`나 `XDP_DROP`을 반환할 수 있다. 같은 Code를 Tracepoint Program으로 Load하면 Tracepoint Context에는 XDP의 `data`, `data_end` Field가 없고 반환값도 XDP Action으로 해석되지 않는다.

```text
XDP Program
  입력: struct xdp_md
  가능한 처리: Packet 범위 검사, Drop·Redirect
  반환: XDP_PASS, XDP_DROP 등

Tracepoint Program
  입력: 해당 Trace Event 전용 Context
  가능한 처리: Event 정보 읽기, 통계·Event 기록
  반환: XDP Action이 아님
```

즉 “C 문법이 맞는가?”와 “현재 Program Type의 Context와 Helper 계약에 맞는가?”는 서로 다른 문제다.

[Linux Kernel BPF Program Types](https://docs.kernel.org/bpf/programs.html), [Cilium BPF Program Types Reference](https://docs.cilium.io/en/stable/reference-guides/bpf/progtypes/)

---



## 6. Verifier Deep Dive

Verifier는 eBPF가 Kernel Context에서 실행될 수 있게 만드는 핵심 안전장치다. Program을 실제 Traffic에서 실행하기 전에 Instruction과 가능한 상태를 정적으로 분석한다.

### 6.1 검증의 큰 흐름

```text
User Space Loader
      │
      │ bpf(BPF_PROG_LOAD, ...)
      ▼
┌──────────────────────────────┐
│ 1. 기본 구조 검사            │
│ Instruction·Control Flow가   │
│ 유효하고 종료 가능한가?       │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ 2. 상태를 추상적으로 실행     │
│ Register Type·값 범위와       │
│ Stack 초기화 상태를 추적      │
└──────────────┬───────────────┘
               ▼
┌──────────────────────────────┐
│ 3. 경로별 안전성 검사         │
│ Branch, Pointer, Boundary,    │
│ Helper Argument를 검증        │
└──────────────┬───────────────┘
               ▼
       모든 경로를 증명했는가?
          ┌────┴────┐
        Yes         No
         │           │
         ▼           ▼
      Accept      Reject
  Program FD 반환  Verifier Log 반환
```

대표적으로 다음을 확인한다.

- Program이 유효한 Control Flow를 가지고 종료되는가?
- 초기화하지 않은 Register와 Stack Slot을 읽지 않는가?
- Packet이나 Map Value의 범위를 벗어나지 않는가?
- Pointer의 Type과 Offset 연산이 유효한가?
- 현재 Program Type에서 허용한 Helper만 사용하는가?
- Helper 호출에 전달한 Argument Type이 올바른가?
- 실행 경로와 Instruction 복잡도를 Kernel이 분석할 수 있는가?
- 반복이 있다면 실행 범위를 분석하고 종료 가능성을 판단할 수 있는가?



### 6.2 Verifier는 값을 추상 상태로 추적한다

Verifier는 Program을 실제 Packet 하나나 특정 값 하나로 실행하지 않는다. 대신 각 Register가 가질 수 있는 값의 종류와 가능한 범위를 추적한다. 이를 추상 상태라고 한다.

```text
R1 = PTR_TO_CTX
R2 = SCALAR_VALUE, 범위 0~1500
R3 = PTR_TO_PACKET
R4 = PTR_TO_MAP_VALUE_OR_NULL
R10 = Read-only Frame Pointer
```

| 추상 상태 | 의미 | 허용되는 접근의 예 |
| --- | --- | --- |
| `PTR_TO_CTX` | Program Type이 제공한 Context를 가리키는 Pointer | 허용된 Context Field 읽기 |
| `SCALAR_VALUE` | Pointer가 아닌 정수 값 | 산술·비교. 알려진 최소·최대 범위도 함께 추적 |
| `PTR_TO_PACKET` | Packet Data 안의 위치를 가리키는 Pointer | `data_end`로 안전 범위를 증명한 뒤 Packet 읽기 |
| `PTR_TO_MAP_VALUE_OR_NULL` | Map Value Pointer이거나 Entry가 없음을 뜻하는 NULL | NULL Check 전에는 Value 접근 불가 |
| `PTR_TO_MAP_VALUE` | NULL이 아님을 증명한 Map Value Pointer | Map에 선언된 Value 크기 안에서 읽기·쓰기 |
| `PTR_TO_STACK` | 현재 eBPF Stack을 가리키는 Pointer | Stack 범위 안에서 초기화된 영역 접근 |

예시의 `R2 = SCALAR_VALUE, 범위 0~1500`은 실제 값은 모르지만 0보다 작거나 1500보다 클 수 없다는 뜻이다. Verifier는 이런 범위 정보로 Pointer Offset이나 Memory 접근이 안전한지 판단한다.

예를 들어 Packet Pointer에 검증되지 않은 큰 값을 더하면 Packet 끝을 넘어갈 수 있다. Verifier가 안전 범위를 증명할 수 없다면 Program을 거부한다.

### 6.3 Branch는 상태를 더 구체적으로 만든다

Verifier는 `if` 같은 Branch를 만나면 두 실행 경로를 나누어 분석한다. 중요한 점은 조건 검사가 단순히 실행 방향만 정하는 것이 아니라, 각 경로에서 참인 사실을 Verifier에게 알려 준다는 것이다. “이 Pointer가 NULL이 아니다”, “이 Header가 Packet 끝을 넘지 않는다” 같은 사실은 검사 이후의 안전한 Memory 접근 근거가 된다.

다음 XDP 형태의 Code에서는 Ethernet Header를 읽기 전에 Packet 안에 Header 전체가 들어 있는지 Branch로 확인한다.

```c
void *data = (void *)(long)ctx->data;
void *data_end = (void *)(long)ctx->data_end;
struct ethhdr *eth = data;

if ((void *)(eth + 1) > data_end)
    return XDP_DROP;

/* 이 경로에서는 eth Header 전체가 Packet 범위 안에 있음 */
```

분기 전에는 `eth + 1`이 `data_end`를 넘을 수 있다. 조건을 통과한 경로에서는 Ethernet Header 크기만큼 접근해도 Packet 범위 안이라는 정보가 생긴다.

```text
분기 전
  eth Pointer의 최소 범위만 알고 있음

if eth + 1 > data_end
  ├─ True  → Header가 부족함 → DROP
  └─ False → Header 전체가 범위 안 → 접근 허용
```

Boundary Check를 빼면 Compiler 관점에서는 유효한 C Code여도 Verifier는 안전성을 증명할 수 없어 Load를 거부할 수 있다.

### 6.4 NULL Check도 상태 검증이다

Map Lookup은 Entry가 없을 때 `NULL`을 반환할 수 있다.

```c
value = bpf_map_lookup_elem(&map, &key);
if (!value)
    return XDP_PASS;

*value += 1;
```

```text
Lookup 직후
  value = PTR_TO_MAP_VALUE_OR_NULL

NULL Check 통과
  value = PTR_TO_MAP_VALUE

그 이후
  Map Value 범위 안에서 접근 가능
```

NULL Check 없이 `*value`에 접근하면 NULL Pointer 가능성을 배제할 수 없으므로 거부된다.

### 6.5 State Pruning

분기가 많을수록 가능한 실행 상태가 급격히 늘어난다. Verifier는 같은 Instruction에 이미 도달했던 더 일반적이고 안전한 상태가 있다면 현재 경로를 다시 끝까지 분석하지 않고 줄일 수 있다.

```text
여러 Branch
  → 같은 Instruction에 도착
  → 기존에 검증한 State가 현재 State를 포함하는지 비교
  → 포함한다면 해당 경로의 중복 분석 제거
```

이를 State Pruning이라고 한다. Program이 너무 복잡하거나 State가 충분히 합쳐지지 않으면 안전한 Logic이라도 Verifier의 복잡도 제한에 걸릴 수 있다.

### 6.6 Verifier가 보장하지 않는 것

Verifier 통과는 Program의 기능이 올바르다는 뜻이 아니다.

```text
Verifier가 확인
  → Kernel Memory를 위험하게 사용하지 않는가?
  → 허용된 실행 계약을 지키는가?

Verifier가 확인하지 않음
  → 운영자가 의도한 Packet만 Drop하는가?
  → Counter 계산이 Business 요구사항과 맞는가?
  → Policy Logic에 논리적 실수가 없는가?
```

예를 들어 모든 Packet을 `XDP_DROP`하는 Program은 서비스 관점에서는 장애를 만들지만, eBPF 안전 규칙을 지키면 Verifier를 통과할 수 있다.

[Linux Kernel eBPF Verifier](https://docs.kernel.org/bpf/verifier.html)

---



## 7. Helper: 제한된 Kernel 기능의 입구

eBPF Program은 임의의 Kernel Function을 마음대로 호출하지 않는다. Kernel이 eBPF에 공개한 Helper를 통해 필요한 기능을 요청한다.

대표적인 기능은 다음과 같다.

- BPF Map Lookup, Update, Delete
- 현재 시간, CPU, Process 정보 조회
- Packet Redirect
- Checksum 갱신
- Event를 User Space로 전달
- Tail Call 실행

```text
eBPF Program
  → R1~R5에 Argument 준비
  → Helper Call
  → Kernel이 Argument와 Program Type에 맞는 Helper 실행
  → R0로 결과 반환
```

모든 Helper를 모든 Program Type에서 사용할 수 있는 것은 아니다. Packet을 Redirect하는 Helper는 Network Context가 필요하고, 특정 Tracing Helper는 Tracing Program에서만 허용될 수 있다.

Verifier는 다음을 함께 검사한다.

```text
현재 Program Type에서 호출 가능한 Helper인가?
  → Argument Register의 Type은 맞는가?
  → 전달한 Buffer의 크기와 Memory 영역은 유효한가?
  → Helper 호출 뒤 Pointer 상태가 바뀌는가?
```

즉 Helper는 편의 함수일 뿐 아니라, eBPF Program이 Kernel 기능에 접근할 수 있도록 만든 검증 가능한 API 경계다. [Linux Kernel BPF Helpers](https://docs.kernel.org/bpf/helpers.html)

---



## 8. BPF Map Deep Dive

BPF Map은 eBPF Program과 User Space가 함께 사용할 수 있는 Kernel 내부 자료구조다. 일반 변수와 달리 Program 호출이 끝난 뒤에도 값을 유지할 수 있으며, Map Type에 따라 Hash Table, Array, LRU Cache, Event Buffer 또는 다른 Program을 가리키는 Table처럼 동작한다.

Map을 만들 때 다음 속성을 정한다.

```text
Map Type       → 자료구조와 동작 방식
Key Size       → Entry를 찾는 Key의 크기
Value Size     → Entry 하나에 저장할 값의 크기
Max Entries    → 저장할 수 있는 최대 Entry 수
Flags          → 생성·접근 방식에 대한 추가 Option
```

여기서 Entry는 Map 안에 저장된 한 개의 항목을 뜻한다. Hash Map의 Entry는 `Key → Value` 한 쌍이고, Array Map의 Entry는 정수 Index에 대응하는 Value 한 칸이다.

### 8.1 Program 호출 사이에서 상태를 유지한다

eBPF Program의 Stack과 Register는 한 번의 호출이 끝나면 다음 Event에서 그대로 유지되지 않는다. 지속적인 상태는 BPF Map에 둔다.

```text
Event 1 → Program 실행 ─┐
Event 2 → Program 실행 ─┼─► BPF Map
Event 3 → Program 실행 ─┘

User Space Controller ─────► 같은 Map 조회·갱신
```

Map은 Kernel에 존재하는 Object이며 Key와 Value 크기, 최대 Entry 수, Map Type 같은 속성을 가진다.

### 8.2 File Descriptor와 Pinning

User Space는 `bpf()` System Call로 Map을 만들고 File Descriptor를 통해 접근한다.

```text
BPF_MAP_CREATE
  → Kernel Map Object 생성
  → User Space에 File Descriptor 반환
```

`BPF_MAP_CREATE`는 Map 내부에서 실행하는 명령이 아니다. 하나의 `bpf()` System Call이 여러 eBPF 작업을 지원하므로, 첫 번째 Argument인 `cmd`로 어떤 작업을 요청하는지 구분하는 Subcommand 값이다.

```c
fd = bpf(BPF_MAP_CREATE, &attr, sizeof(attr));
```

개념적으로 `attr`에는 Map Type, Key·Value 크기와 최대 Entry 수 같은 생성 정보가 들어간다. 호출이 성공하면 반환된 FD를 이용해 User Space에서 Map을 조회·갱신하거나 다른 BPF Object와 연결할 수 있다. `libbpf` 같은 Library를 사용하면 보통 이 System Call을 직접 조립하지 않고 Library API를 호출한다.

열린 File Descriptor나 Program 같은 참조가 모두 사라지면 Map도 정리될 수 있다. BPF File System에 Map을 Pin하면 경로를 통해 다시 열 수 있고, 처음 만든 Process의 생명주기와 분리할 수 있다.

```text
/sys/fs/bpf/example_map
  → 다른 Process가 Path를 통해 Map 참조
  → Loader가 종료된 뒤에도 참조 유지 가능
```

`/sys/fs/bpf`는 보통 BPF File System(bpffs)이 Mount되는 경로이고, `example_map`은 사용자가 붙인 Pin 이름의 예다. Pinning은 Kernel Object의 내용을 일반 File로 저장한다는 뜻이 아니라, bpffs의 Path가 그 Object에 대한 Reference를 유지하게 하는 방식이다.

```text
Map FD 7 ───────► Kernel Map Object
                       ▲
bpffs Path ────────────┘
/sys/fs/bpf/example_map

Loader 종료로 FD 7이 닫힘
  → Path Reference가 남아 있으므로 Map 유지

Path를 Unpin하고 다른 Reference도 없음
  → Kernel이 Map 정리 가능
```



### 8.3 대표 Map Type


| Map Type                       | 특징                             | 대표 사용 사례               |
| ------------------------------ | ------------------------------ | ---------------------- |
| Hash                           | Key의 정확한 일치                    | Flow, PID, IP별 상태      |
| LRU Hash                       | 공간이 부족하면 오래 사용하지 않은 Entry부터 제거 | 제한된 크기의 Connection 상태  |
| Array                          | 정수 Index와 고정된 Entry 수          | 설정, Histogram, Counter |
| Per-CPU Hash/Array             | CPU마다 별도 Value 보관              | Lock 경쟁을 줄인 통계         |
| LPM Trie                       | 가장 긴 Prefix 일치                 | IP/CIDR Lookup         |
| Ring Buffer / Perf Event Array | Kernel Event를 User Space로 전달   | Tracing과 Observability |
| Program Array                  | Program File Descriptor 저장     | Tail Call 대상 선택        |
| Map-in-Map                     | 다른 Map의 참조 저장                  | 상태 집합의 교체와 분리          |




### 8.4 Per-CPU Map이 필요한 이유

여러 CPU에서 매우 자주 하나의 Counter를 갱신하면 Cache Line 경쟁과 동기화 비용이 발생할 수 있다.

```text
Global Counter

CPU 0 ─┐
CPU 1 ─┼─► 같은 Value 갱신
CPU 2 ─┘    → 동기화와 Cache 경쟁
```

Per-CPU Map은 CPU마다 별도 Value를 둔다.

```text
CPU 0 → Counter[CPU 0]
CPU 1 → Counter[CPU 1]
CPU 2 → Counter[CPU 2]

User Space 조회
  → CPU별 값을 합산
```

Write 경쟁을 줄일 수 있지만, 전체 값을 읽을 때 CPU별 값을 합쳐야 하고 CPU Hotplug와 Memory 사용량도 고려해야 한다.

### 8.5 LRU Map의 의미

Connection Tracking처럼 Entry가 계속 늘 수 있는 상태를 무한히 보관할 수는 없다. 여기서 Entry는 Source·Destination IP와 Port 같은 Key 하나와, 연결 상태·마지막 사용 시각 같은 Value 하나로 이루어진 Map 항목이다. LRU Hash는 최대 크기에 도달하면 오래 사용하지 않은 Entry를 제거해 새 Entry를 넣을 공간을 만든다.

```text
Map Full
  → 오래 사용하지 않은 Entry 선택
  → 기존 Entry 제거
  → 새 Entry 삽입
```

LRU는 편리하지만 “중요하지 않은 Entry”가 아니라 “최근에 사용하지 않은 Entry”를 제거한다. Workload 특성에 따라 아직 필요한 상태가 밀려날 가능성과 Map 크기를 함께 고려해야 한다.

### 8.6 Map에는 동시성이 존재한다

여러 CPU의 Program과 User Space가 같은 Map을 동시에 읽고 쓸 수 있다. Map Operation 하나가 안전하다는 것과 여러 Operation으로 구성된 Logic 전체가 원자적이라는 것은 다르다.

```text
lookup → value 계산 → update
```

이 사이에 다른 CPU가 같은 값을 바꿀 수 있다. 필요에 따라 Atomic Operation, Spin Lock을 지원하는 Value, Per-CPU Map 또는 Logic 재설계를 사용해야 한다.

[Linux Kernel BPF Maps](https://docs.kernel.org/bpf/maps.html), [Cilium BPF Architecture: Maps](https://docs.cilium.io/en/stable/reference-guides/bpf/architecture/#maps)

---



## 9. BPF-to-BPF Call과 Tail Call

복잡한 eBPF Logic을 여러 부분으로 나누는 대표 방법에는 Function Call과 Tail Call이 있다.

### 9.1 BPF-to-BPF Function Call

```text
Program Main
  → BPF Function 호출
  → Function 실행
  → 호출한 위치로 복귀
  → Main 계속 실행
```

일반적인 Function Call과 비슷하다. 같은 Program Object 안에서 Code를 재사용하고 Logic을 나누는 데 사용할 수 있다.

### 9.2 Tail Call

Tail Call의 `tail`은 Function의 마지막 동작으로 다른 Program에 실행을 넘기고 원래 호출자로 돌아오지 않는다는 의미다. Program을 꼬리처럼 여러 단계로 연결할 수 있지만, 핵심은 “이어진 모양”보다 “복귀하지 않는 Jump”에 있다.

Tail Call은 `BPF_MAP_TYPE_PROG_ARRAY`에서 다음 Program을 찾아 실행을 넘긴다.

```text
Program A
  → Program Array에서 Index 조회
  → bpf_tail_call() Helper 호출
  → Program B
  → Program A로 돌아오지 않음
```

`bpf_tail_call()`은 User Space가 Kernel에 요청하는 System Call이 아니라, 실행 중인 eBPF Program이 호출하는 BPF Helper다. `bpf()` System Call과 이름이 비슷하지만 호출 위치와 목적이 다르다.

```text
User Space
  bpf(BPF_PROG_LOAD, ...)
  → Program을 Kernel에 Load하는 System Call

eBPF Program 내부
  bpf_tail_call(ctx, &programs, index)
  → 다음 eBPF Program으로 Jump하는 Helper
```

```text
Function Call
  A → B → A → 계속 실행

Tail Call 성공
  A → B → B에서 계속 실행
```

Tail Call 대상은 호환되는 Program Type 등의 조건을 만족해야 하며, 무한한 Program 연결을 막기 위한 호출 횟수 제한도 있다. Target을 찾지 못하거나 Tail Call이 실패하면 현재 Program의 다음 Instruction을 계속 실행한다.

### 9.3 왜 Tail Call을 사용하는가

```text
하나의 거대한 Program
  Parse + Policy + NAT + Routing + Event
```

```text
단계별 Program
  Parse
    → Policy
      → NAT
        → Routing
          → Event
```

- 기능별로 Program을 나눌 수 있음
- Program Array의 Entry를 바꿔 Runtime에 다음 단계를 선택 가능
- 선택하지 않은 기능의 Code를 항상 실행하지 않아도 됨
- 개별 Program 단위로 Verifier의 검증을 받음

예를 들어 Program Array의 Index `0`을 IPv4 처리 Program, Index `1`을 IPv6 처리 Program에 연결할 수 있다. Entry의 Program FD를 새 Version으로 교체하면 첫 Program을 다시 Load하지 않고도 이후 Packet이 새 처리 Program으로 이동하게 만들 수 있다.

```text
Packet Parser
  ├─ IPv4 → programs[0] → IPv4 Policy v1
  └─ IPv6 → programs[1] → IPv6 Policy

User Space가 programs[0]의 Entry 교체

Packet Parser
  ├─ IPv4 → programs[0] → IPv4 Policy v2
  └─ IPv6 → programs[1] → IPv6 Policy
```

다만 Tail Call 대상 Program은 호출한 Program의 지역 변수나 Register 값을 이어받아 사용할 수 없다. 예를 들어 Program A의 Stack에 `is_trusted = true`라는 지역 변수를 저장해도 Program B가 그 변수를 직접 읽을 수 없으며, Tail Call 성공 뒤에는 A로 돌아와 후속 Code를 실행할 수도 없다.

```text
Program A의 지역 상태
  R6 = 임시 Pointer
  Stack[-8] = is_trusted
       │
       └─ Tail Call 성공 ─► Program B에서 직접 접근 불가

공유가 필요한 상태
  → 원래 Context
  → Program Type이 허용하는 Context Metadata
  → Per-CPU Map 같은 BPF Map
```

즉 Kernel은 Tail Call에 같은 실행 자원을 재사용할 수 있지만, 개발자 관점에서는 일반 Function Call처럼 Caller의 Stack Frame으로 돌아오거나 지역 상태를 공유하는 구조가 아니다.

[Cilium BPF Architecture: Tail Calls](https://docs.cilium.io/en/stable/reference-guides/bpf/architecture/#tail-calls)

---



## 10. JIT와 실행 성능

JIT(Just-In-Time Compilation)는 Program을 실제로 실행할 시점에 가까워졌을 때 중간 형태의 Code를 현재 CPU가 직접 실행할 수 있는 Native Instruction으로 변환하는 방식이다. eBPF에서는 Load된 Bytecode를 x86-64나 arm64 같은 Node의 Architecture에 맞게 바꿔 Interpreter가 Instruction을 하나씩 해석하는 비용을 줄일 수 있다.

### 10.1 Interpreter와 JIT

Verifier를 통과한 eBPF Bytecode는 Kernel Interpreter가 실행하거나 JIT Compiler가 현재 CPU의 Native Instruction으로 변환할 수 있다.

```text
eBPF Bytecode
  ├─ Interpreter
  │    → Bytecode Instruction을 해석하며 실행
  │
  └─ JIT
       → x86-64, arm64 등의 Native Instruction 생성
       → 생성한 Code 실행
```

동일한 eBPF Bytecode도 Node의 Architecture에 따라 다른 Native Code가 만들어진다.

```text
eBPF Bytecode
  ├─ x86-64 Node → x86-64 Instruction
  └─ arm64 Node  → arm64 Instruction
```

JIT 사용 가능 여부와 동작은 Kernel Build, Architecture와 보안 설정에 영향을 받는다. 따라서 개념 흐름에서는 `Verifier → JIT`로 단순하게 그릴 수 있지만, JIT가 모든 환경의 필수 단계는 아니다.

### 10.2 eBPF의 성능을 JIT 하나로 설명할 수 없는 이유

eBPF 사용이 성능상 유리할 수 있는 이유는 여러 가지다.

- Event가 발생한 Kernel Hook에서 바로 처리 가능
- User Space와 Kernel 사이의 반복적인 전환과 Data 전달 감소
- Hash, LPM Trie, Per-CPU Map처럼 목적에 맞는 자료구조 사용
- XDP처럼 Network Stack의 이른 지점에서 불필요한 Packet Drop 가능
- 필요한 기능만 Program에 포함하거나 여러 Program으로 분리 가능
- JIT를 사용하면 Native Instruction으로 실행 가능

반대로 Program이 복잡하거나 Map Lookup과 Event 출력이 많으면 비용이 커질 수 있다. Hook이 이르다는 이유만으로 Application 전체가 빨라지는 것도 아니다.

### 10.3 성능을 평가할 때 관찰 비용도 포함해야 한다

이 내용이 실행 성능 절에 포함되는 이유는 eBPF Program 자체의 처리 시간만으로 전체 비용을 판단할 수 없기 때문이다. Observability Program은 Event를 수집한 뒤 User Space로 전달하고 해석·저장하는 경우가 많다. 따라서 “Kernel Hook에서 빠르게 실행됐다”는 사실과 “관찰 시스템 전체가 저렴하다”는 결론은 같지 않다.

Packet마다 큰 Event를 User Space로 보내면 다음 비용이 생긴다.

```text
eBPF Program
  → Event 생성
  → Buffer 기록
  → User Space Wake-up / Poll
  → Decode와 저장
```

필요한 Event만 Sampling하거나 Kernel에서 먼저 집계한 뒤 요약 값을 Map으로 읽는 방법을 고려할 수 있다. “Kernel에서 실행되므로 항상 비용이 거의 없다”는 설명은 정확하지 않다.

[Linux Kernel Socket Filtering and BPF JIT](https://docs.kernel.org/networking/filter.html), [Cilium BPF Architecture](https://docs.cilium.io/en/stable/reference-guides/bpf/architecture/)

---

## 11. 작은 예제로 전체 과정 연결하기

목적지 TCP Port가 8080인 XDP Packet의 수를 세고 나머지는 통과시키는 개념적인 Program을 생각해 본다.

```c
SEC("xdp")
int count_tcp_8080(struct xdp_md *ctx)
{
    void *data = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;
    struct ethhdr *eth = data;

    if ((void *)(eth + 1) > data_end)
        return XDP_DROP;

    /* 실제 구현에서는 VLAN, IPv4 Header 길이, Fragment 등을 더 처리해야 함 */

    if (destination_port_is_8080(data, data_end)) {
        __u32 key = 0;
        __u64 *count = bpf_map_lookup_elem(&counter, &key);
        if (count)
            __sync_fetch_and_add(count, 1);
    }

    return XDP_PASS;
}
```

이 예시는 흐름을 설명하기 위한 Pseudo Code이며 그대로 Compile할 수 있는 완전한 Program은 아니다.

### 11.1 Compile

```text
C Source
  → clang -target bpf
  → eBPF Instruction과 Map 정의가 포함된 Object
```



### 11.2 Load와 Verify

Verifier는 다음을 확인한다.

```text
ctx가 XDP Context인가?
  → data와 data_end 접근이 허용되는가?
  → Ethernet Header Boundary Check가 있는가?
  → Map Lookup Helper Argument가 올바른가?
  → count가 NULL이 아닌 경로에서만 값을 쓰는가?
  → 마지막 반환값이 XDP Action으로 유효한가?
```



### 11.3 Attach

Program을 특정 Network Interface의 XDP Hook에 연결한다.

```text
NIC eth0
  → XDP Hook
  → count_tcp_8080 Program
```



### 11.4 Execute

```text
Packet 수신
  → Driver의 XDP Hook
  → ctx로 Packet 범위 전달
  → Header Boundary Check
  → TCP 8080이면 Counter Map Lookup과 증가
  → XDP_PASS 반환
  → Packet이 일반 Network Stack으로 계속 이동
```



### 11.5 User Space에서 상태 조회

Loader나 Debug 도구가 Counter Map을 읽으면 Program을 다시 실행하지 않고도 누적 값을 확인할 수 있다.

```text
bpftool map lookup ...
  → CPU별 또는 Global Counter 확인
```

이 작은 예제 안에 Program Type, Context, Verifier, Helper, Map, Hook과 반환값이 모두 들어 있다.

---



## 12. 선택 실습: `bpftool`로 Kernel Object 확인



### 12.1 Program 확인

```bash
sudo bpftool prog show
```

확인할 항목:

- Program ID와 Name
- Program Type
- Load 시간과 UID
- JIT된 Size 또는 Instruction 관련 정보
- Program이 사용하는 Map ID



### 12.2 Map 확인

```bash
sudo bpftool map show
```

확인할 항목:

- Map ID와 Name
- Map Type
- Key와 Value Size
- 최대 Entry 수
- 현재 Pin 경로 여부

특정 Map의 Entry는 다음 형태로 확인한다.

```bash
sudo bpftool map dump id <map-id>
```



### 12.3 Attach 지점 확인

```bash
# XDP와 TC 등 Network Attach 상태
sudo bpftool net show

# cgroup v2 아래에 Attach된 Program
sudo bpftool cgroup tree /sys/fs/cgroup
```



### 12.4 JIT 결과 확인

Kernel 설정과 권한이 허용하면 Program의 JIT 결과를 확인할 수 있다.

```bash
sudo bpftool prog dump jited id <program-id>
```

비교 질문:

1. eBPF Instruction 수와 JIT된 Native Instruction 크기는 어떤 관계인가?
2. 같은 역할의 Program도 Program Type에 따라 Context와 반환값이 어떻게 다른가?
3. Program이 참조하는 Map은 어떤 Type이며 왜 그 Type을 선택했는가?

> 실제 출력과 지원 Option은 `bpftool` 및 Kernel Version에 따라 달라질 수 있으므로 설치 환경의 `bpftool help`도 함께 확인한다.

---



## 13. Cilium과 Hubble은 eBPF를 어떻게 활용하는가

지금까지 살펴본 eBPF 구성요소를 실제 Kubernetes Network 구현에 적용한 사례가 Cilium이다.


| eBPF 개념                      | Cilium에서의 활용 예시                                        |
| ---------------------------- | ------------------------------------------------------ |
| User Space Loader/Controller | `cilium-agent`가 Program과 Map 관리                        |
| TC Program                   | Pod Interface의 Policy, Forwarding과 Network 처리          |
| cgroup/socket Program        | Socket Load Balancing 등                                |
| XDP Program                  | 설정에 따른 빠른 Service 처리와 Drop                             |
| Hash/LPM/Per-CPU Map         | Endpoint, Identity, Policy, Service, Connection과 통계 상태 |
| Event 출력                     | Data Path의 전달·Drop 정보를 User Space로 전달                  |


```text
Kubernetes API
  → cilium-agent
      ├─ eBPF Program Load / Attach
      └─ BPF Map Update
            │
            ▼
Packet → Kernel Hook → eBPF Program → Map Lookup → Forward / Drop
                                               └→ Event
                                                    │
                                                    ▼
                                                  Hubble
```

여기서 중요한 것은 Cilium의 각 Map 이름을 외우는 것이 아니다.

```text
eBPF Program  = Packet 처리 Logic
BPF Map       = 현재 Kubernetes Network 상태
cilium-agent  = Logic과 상태를 Kernel에 반영하는 User Space Controller
Hubble        = Data Path Event를 Workload Context와 함께 보여주는 관측 계층
```

Hubble은 Packet을 별도의 User Space Proxy로 통과시켜 다시 판단하는 구조가 아니다. Cilium Data Path에서 만들어진 Event를 수집하고 Pod, Namespace, Identity, Verdict 같은 정보를 연결해 Flow로 제공한다.

즉 Cilium과 Hubble은 별도의 새로운 원리가 아니라, 앞에서 살펴본 Program, Map, Hook과 Event 출력 모델을 Kubernetes Networking에 적용한 사례로 이해할 수 있다.

[Cilium Component Overview](https://docs.cilium.io/en/stable/overview/component-overview/), [Hubble Internals](https://docs.cilium.io/en/stable/internals/hubble/)

---



## 14. 자주 생기는 오해



### 오해 1. eBPF Program은 Kernel에서 계속 실행되는 Process다

아니다. 특정 Hook의 Event가 발생할 때 호출되어 작업을 수행하고 반환한다. 지속적인 제어와 상태 갱신은 보통 User Space Controller와 Map이 담당한다.

### 오해 2. Verifier를 통과하면 Program Logic도 올바르다

아니다. Verifier는 Memory 접근과 실행 계약의 안전성을 검증한다. 잘못된 Packet을 Drop하는 논리 오류까지 판단하지 않는다.

### 오해 3. 현대 eBPF에서는 Loop를 전혀 사용할 수 없다

아니다. Kernel과 Program 형태에 따라 Bounded Loop, Helper 기반 반복, Open-coded Iterator 등을 사용할 수 있다. 다만 Verifier가 실행 범위와 안전성을 분석할 수 있어야 한다.

### 오해 4. eBPF Program은 모든 Kernel Function을 호출할 수 있다

아니다. Program Type에 허용된 Helper와 조건을 만족하는 Kernel Function만 사용할 수 있으며 Verifier가 Argument와 호출 가능 여부를 검사한다.

### 오해 5. Map Lookup은 일반 Pointer Lookup과 같다

아니다. Map Lookup 결과는 `NULL`일 수 있고, Verifier가 추적하는 Map Value Pointer다. NULL Check와 Value 범위 안의 접근이 필요하다.

### 오해 6. XDP가 가장 이른 Hook이므로 모든 Network 기능은 XDP에 구현해야 한다

아니다. XDP는 빠르지만 `skb`와 상위 Network Metadata가 아직 없다. 필요한 기능과 Context에 따라 TC, Socket 등 다른 Hook이 더 적합할 수 있다.

### 오해 7. eBPF는 JIT 덕분에 항상 기존 방식보다 빠르다

아니다. JIT는 중요한 요소지만 Hook 위치, Program 복잡도, Map 경쟁, Event 출력과 Workload 특성이 함께 성능을 결정한다.

---



## 15. 학습하면서 생긴 질문



### Q1. C Source에는 문제가 없어 보이는데 Verifier가 거부하는 이유는 무엇일까?

Verifier가 보는 것은 Source가 아니라 Compiler가 만든 Instruction과 가능한 실행 상태다. 특정 Branch에서 Boundary Check가 적용되지 않거나, Helper 호출로 Pointer 유효성이 바뀌거나, Program의 State 수가 너무 복잡해 안전성을 증명하지 못할 수 있다. Verifier Log와 실제 eBPF Instruction을 함께 확인해야 한다.

### Q2. Program과 Map을 Pin하지 않은 상태에서 Loader가 종료되면 어떻게 될까?

Kernel Object는 Reference가 남아 있는 동안 유지된다. Attach Link, 다른 File Descriptor 같은 Reference도 없고 BPF File System에 Pin하지 않았다면 Program과 Map이 정리될 수 있다. 사용하는 Attach 방식과 Object의 Reference 관계를 함께 확인해야 한다.

### Q3. Per-CPU Map을 사용하면 동시성 문제를 모두 해결할 수 있을까?

아니다. 같은 CPU의 값에 대한 처리와 전체 값의 합산, CPU별 Memory 사용, CPU Hotplug를 고려해야 한다. 여러 CPU가 반드시 하나의 일관된 값을 공유해야 하는 상태라면 Per-CPU 분리는 요구사항과 맞지 않을 수 있다.

Packet 수를 통계로 집계하는 경우에는 약간 늦게 합산해도 되므로 Per-CPU Counter가 잘 맞는다.

```text
CPU 0 = 120 packets
CPU 1 = 95 packets
CPU 2 = 85 packets

User Space가 조회할 때 합산
  → 총 300 packets
```

반면 “전체 Cluster에서 동시에 허용할 Connection은 최대 100개”처럼 모든 CPU가 같은 순간에 하나의 제한을 지켜야 하는 경우는 다르다. CPU별로 100개씩 허용하면 전체 제한을 초과하고, User Space가 나중에 합산해도 이미 발생한 초과를 되돌릴 수 없다. 이런 상태에는 Global Counter와 적절한 Atomic·Lock 전략 또는 다른 제어 구조가 필요하다.

### Q4. Tail Call로 Program을 계속 연결하면 Verifier의 크기 제한을 피할 수 있을까?

Program을 나눠 각 Program을 독립적으로 검증받을 수는 있지만, Tail Call 횟수와 호환 조건이 있고 상태 전달에도 제약이 있다. 제한을 우회하는 수단보다 Data Path를 기능 단위로 구성하는 Mechanism으로 이해하는 편이 정확하다.

---



## 16. 핵심 용어 정리


| 용어               | 설명                                                                     |
| ---------------- | ---------------------------------------------------------------------- |
| eBPF Instruction | eBPF Virtual Machine이 실행하는 Instruction Set의 명령                         |
| ELF Object       | eBPF Instruction, Map 정의, BTF와 Section 정보를 담아 Loader에 전달하는 Object File |
| Loader           | Object를 열고 Program과 Map을 Kernel에 Load하는 User Space 구성요소                |
| `bpf()` System Call | User Space가 Program·Map 생성과 조회 같은 eBPF 작업을 Kernel에 요청하는 Interface       |
| Program Type     | Context, Helper, 반환값과 Attach 가능한 Hook의 계약을 결정하는 분류                     |
| Hook             | 특정 Kernel Event에서 eBPF Program이 호출되도록 연결하는 지점                          |
| Context          | Hook이 Program에 입력으로 전달하는 Packet, Socket, Trace 등의 정보                   |
| Verifier         | Load 전에 Instruction의 실행 상태와 Memory 안전성을 검사하는 Kernel 구성요소               |
| Helper           | eBPF Program에 공개된 검증 가능한 Kernel 기능의 호출 경계                              |
| BPF Map          | Program 호출 사이의 상태와 User/Kernel 공유 Data를 저장하는 Kernel Object             |
| Entry            | Map 안에 저장된 Key와 Value 한 쌍 또는 Array의 한 Index에 해당하는 항목                  |
| bpffs            | Pinning을 위한 Path를 제공하는 BPF 전용 File System                              |
| Pinning          | BPF File System의 Path로 Program이나 Map의 Reference를 유지하는 방식               |
| Per-CPU Map      | CPU마다 별도 Value를 보관해 경쟁을 줄일 수 있는 Map 형태                                 |
| BPF-to-BPF Call  | 실행 뒤 호출한 위치로 복귀하는 eBPF Function Call                                   |
| Tail Call        | Program Array를 조회해 다른 Program으로 실행을 넘기고 돌아오지 않는 호출                     |
| JIT              | eBPF Bytecode를 현재 CPU Architecture의 Native Instruction으로 변환하는 Compiler |


---



## 17. 참고 자료



### Linux Kernel 공식 문서

- [BPF Instruction Set Architecture](https://docs.kernel.org/bpf/standardization/instruction-set.html)
- [eBPF Verifier](https://docs.kernel.org/bpf/verifier.html)
- [BPF Maps](https://docs.kernel.org/bpf/maps.html)
- [BPF Helper Functions](https://docs.kernel.org/bpf/helpers.html)
- [BPF Program Types](https://docs.kernel.org/bpf/programs.html)
- [Program Types and ELF Sections](https://docs.kernel.org/bpf/libbpf/program_types.html)
- [libbpf Overview](https://docs.kernel.org/bpf/libbpf/libbpf_overview.html)
- [eBPF Syscall](https://docs.kernel.org/userspace-api/ebpf/syscall.html)
- [Socket Filtering and BPF JIT](https://docs.kernel.org/networking/filter.html)

### Compiler 문서

- [LLVM Target-Independent Code Generator: eBPF Backend](https://llvm.org/docs/CodeGenerator.html#the-extended-berkeley-packet-filter-ebpf-backend)



### eBPF 및 활용 문서

- [eBPF Documentation](https://docs.ebpf.io/)
- [Cilium BPF Architecture](https://docs.cilium.io/en/stable/reference-guides/bpf/architecture/)
- [Cilium BPF Program Types](https://docs.cilium.io/en/stable/reference-guides/bpf/progtypes/)
- [Cilium Component Overview](https://docs.cilium.io/en/stable/overview/component-overview/)
- [Hubble Internals](https://docs.cilium.io/en/stable/internals/hubble/)
