# eBPF Deep Dive & Hubble - week 7

## eBPF와 hook

#### **eBPF(extended Berkeley Packet Filter)**

리눅스 커널에서 작은 사용자 정의 코드를 실행할 수 있게 해주는 기술

<img width="773" height="275" alt="image" src="https://github.com/user-attachments/assets/7d234630-ea83-4507-84bb-cdc4e2ae382b" />


**커널의 필요한 지점에서 직접 코드를 실행하여 네트워크 처리와 관찰 기능을 유연하게 구현할 수 있다**

#### **eBPF Map**

eBPF 프로그램이 처리에 필요한 정보를 저장하고 조회하는 커널 내부의 데이터 구조

| 정보 | 예시 |
| --- | --- |
| **Service·backend 정보** | Service 주소 → 실제 backend Pod 목록 |
| **Identity 정보** | Pod IP → Security Identity |
| **정책 정보** | 특정 Identity의 TCP 8080 접근 허용 |
| **연결 상태** | 이미 처리 중인 연결의 상태 |

예를 들어 Service로 연결 요청이 들어오면, **eBPF 프로그램이 Map에서 backend 정보를 조회하여 실제 목적지를 선택**한다.

**eBPF 프로그램이 ‘처리하는 코드’라면, Map은 ‘그 코드가 사용하는 데이터’이다.**

#### **hook**

**프로그램이 실행되는 위치·이벤트**

| Hook | 실행 지점 | 주요 용도 |
| --- | --- | --- |
| **XDP** | 네트워크 드라이버 최하단에 위치(패킷 수신 직후 가장 먼저 실행) | 빠른 패킷 필터링, 선택적인 NodePort·LoadBalancer 가속 |
| **TC ingress/egress** | 네트워크 인터페이스의 수신·송신 지점 | L3/L4 정책 검사, 패킷 변환·전달 |
| **cgroup socket Hook** | `connect()` 등 애플리케이션의 소켓 동작 지점 | Service backend 선택, 소켓 목적지 주소 변경 |

**XDP** : *패킷 처리*

- 네트워크 드라이버 최하단에 위치*(패킷 수신 직후 가장 먼저 실행)*.
- **커널 네트워크 스택 이전에 네트워크 카드 드라이버 레벨에서 직접 런타임 프로그램 실행**.
- 패킷 처리 성능 최상

**TC Ingress/Egress** : *패킷 처리, 정책 적용, 트래픽 redirection*

- 네트워크 인터페이스에 부착*(초기 패킷 처리 이후 L3 레이어 진입 직전 실행)*.
- 패킷과 연관된 metadata에 접근 가능.
- 로컬 노드의
    - **패킷 처리**
    - **L3/L4 정책 적용**
    - **트래픽 redirection**
    - **veth pair의 Host side에 부착되어 컨테이너를 출입하는 모든 트래픽 감시, 정책 강제**에 적합

**sock_ops** : *소켓 레이어 가속화*

- 특정 cgroup에 부착*(TCP 상태 전환 이벤트(`ESTABLISHED`) 발생 시 실행)*.
- (연결된 peer가 동일 로컬 노드/로컬 프록시일 때) **소켓 레이어 가속화** 수행.

**sockmap** : *소켓 통신*

- TCP 소켓 전송(send) 작업 수행 시 실행.
- 메시지 검사 후 drop, 일반 TCP 계층으로 전송 가능
- **네트워크 스택 거치지 않고 다른 소켓으로 메시지 직접 redirect 가능**
- 소켓 통신 가속화

**한 줄 정리: Hook의 이벤트가 발생하면, 그 Hook에 연결된 eBPF 프로그램이 실행되는 것**

## TC Ingress 예제 - `libbpf-bootstrap`

### 1. 컴파일된 eBPF 프로그램 준비

- `tc.bpf.c`**: TC Hook에서 IPv4 패킷의 헤더를 읽어 길이와 TTL을 출력**하고, 패킷 처리를 계속하도록 하는 eBPF 프로그램

```c
// SPDX-License-Identifier: (LGPL-2.1 OR BSD-2-Clause)
/* Copyright (c) 2022 Hengqi Chen */
#include <vmlinux.h>
#include <bpf/bpf_endian.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

#define TC_ACT_OK 0 /* TC의 해당 처리에서 패킷을 허용하고 처리 진행*/
#define ETH_P_IP  0x0800 /* Internet Protocol packet	*/

SEC("tc")
int tc_ingress(struct __sk_buff *ctx)
{
	void *data_end = (void *)(__u64)ctx->data_end; // 패킷 데이터 끝
	void *data = (void *)(__u64)ctx->data; // 패킷 데이터 시작
	struct ethhdr *l2; // Ethernet 헤더 포인터
	struct iphdr *l3; // IPv4 헤더 포인터

	if (ctx->protocol != bpf_htons(ETH_P_IP)) // IPv4가 아니면 바로 통과
		return TC_ACT_OK;

	l2 = data;
	if ((void *)(l2 + 1) > data_end) // Ethernet 헤더를 온전히 읽을 수 없으면 통과
		return TC_ACT_OK;

	l3 = (struct iphdr *)(l2 + 1);
	if ((void *)(l3 + 1) > data_end) // IPv4 기본 헤더를 온전히 읽을 수 없으면 통과
		return TC_ACT_OK;

	bpf_printk("Got IP packet: tot_len: %d, ttl: %d", bpf_ntohs(l3->tot_len), l3->ttl); // 확인한 IPv4 헤더 정보를 출력
	return TC_ACT_OK;
}

char __license[] SEC("license") = "GPL"; // 라이선스 정보를 커널에 전달
```

```c
l2 = data;
if ((void *)(l2 + 1) > data_end) // Ethernet 헤더를 온전히 읽을 수 없으면 통과
	return TC_ACT_OK;

l3 = (struct iphdr *)(l2 + 1);
if ((void *)(l3 + 1) > data_end) // IPv4 기본 헤더를 온전히 읽을 수 없으면 통과
	return TC_ACT_OK;
```

조건이 참이면 **해당 헤더가 현재 직접 읽을 수 있는 범위를 벗어난다는 의미**이다.

이때 발생할 수 있는 상황은 아래와 같다.

- 나머지가 **다른 메모리 영역에 존재**한다.
- 패킷 자체가 짧아 **필요한 데이터가 실제로 없다.**

이 조건문만으로 두 상황을 구별할 수는 없기에 더 읽지 않고 `TC_ACT_OK`를 반환하여 패킷 처리를 커널 네트워크 스택에 맡긴다.

```c
#define TC_ACT_SHOT 2 /* TC의 해당 처리에서 패킷을 차단하고 버린다*/

l2 = data;
if ((void *)(l2 + 1) > data_end)
    return TC_ACT_SHOT;
    
l3 = (struct iphdr *)(l2 + 1);
if ((void *)(l3 + 1) > data_end)
	  return TC_ACT_SHOT;
```

헤더를 온전히 읽을 수 없으면 패킷을 차단하고 버리도록 코드를 수정할 수 있다.

### 2-1. 커널 적재

`tc.c` : 위 `tc.bpf.c` 프로그램을 커널에 적재하고,  선택한 TC Hook에 연결·해제하는 코드

- 커널 적재, Hook - eBPF 연결 작업은 초기에 한 번 수행 이후 대기

```c
// SPDX-License-Identifier: (LGPL-2.1 OR BSD-2-Clause)
/* Copyright (c) 2022 Hengqi Chen */
#include <signal.h>
#include <unistd.h>
#include "tc.skel.h"

#define LO_IFINDEX 1

static volatile sig_atomic_t exiting = 0;

static void sig_int(int signo)
{
	exiting = 1;
}

static int libbpf_print_fn(enum libbpf_print_level level, const char *format, va_list args)
{
	return vfprintf(stderr, format, args);
}

int main(int argc, char **argv)
{
	DECLARE_LIBBPF_OPTS(bpf_tc_hook, tc_hook, .ifindex = LO_IFINDEX,
			    .attach_point = BPF_TC_INGRESS);
	DECLARE_LIBBPF_OPTS(bpf_tc_opts, tc_opts, .handle = 1, .priority = 1);
	bool hook_created = false;
	struct tc_bpf *skel;
	int err;

	libbpf_set_print(libbpf_print_fn);

	skel = tc_bpf__open_and_load(); // eBPF 프로그램을 커널에 적재
	if (!skel) {
		fprintf(stderr, "Failed to open BPF skeleton\n");
		return 1;
	}

	/* The hook (i.e. qdisc) may already exists because:
	 *   1. it is created by other processes or users
	 *   2. or since we are attaching to the TC ingress ONLY,
	 *      bpf_tc_hook_destroy does NOT really remove the qdisc,
	 *      there may be an egress filter on the qdisc
	 */
	err = bpf_tc_hook_create(&tc_hook);
	if (!err)
		hook_created = true;
	if (err && err != -EEXIST) {
		fprintf(stderr, "Failed to create TC hook: %d\n", err);
		goto cleanup;
	}

	tc_opts.prog_fd = bpf_program__fd(skel->progs.tc_ingress); // 연결할 eBPF 프로그램 지정
	err = bpf_tc_attach(&tc_hook, &tc_opts); // Hook과 eBPF 프로그램 연결
	if (err) {
		fprintf(stderr, "Failed to attach TC: %d\n", err);
		goto cleanup;
	}

	if (signal(SIGINT, sig_int) == SIG_ERR) {
		err = errno;
		fprintf(stderr, "Can't set signal handler: %s\n", strerror(errno));
		goto cleanup;
	}

	printf("Successfully started! Please run `sudo cat /sys/kernel/debug/tracing/trace_pipe` "
	       "to see output of the BPF program.\n");

	while (!exiting) {
		fprintf(stderr, ".");
		sleep(1);
	}

	tc_opts.flags = tc_opts.prog_fd = tc_opts.prog_id = 0;
	err = bpf_tc_detach(&tc_hook, &tc_opts);
	if (err) {
		fprintf(stderr, "Failed to detach TC: %d\n", err);
		goto cleanup;
	}

cleanup:
	if (hook_created)
		bpf_tc_hook_destroy(&tc_hook);
	tc_bpf__destroy(skel);
	return -err;
}
```

### 2-2. Verifier 검증을 거친다

- 안전한 메모리 접근을 하도록 작성된 프로그램인지 확인

커널의 프로그램 적재 처리 중 다음 코드가 실행된다

```c
err = bpf_check(&prog, attr, uattr, uattr_size);
if (err < 0)
    goto free_used_maps;
```

| 주요 함수 | 역할 |
| --- | --- |
| `bpf_check()` | 검증을 시작한다. |
| `do_check()` | 명령어와 실행 경로의 상태를 분석한다. |
| `check_mem_access()` | 메모리 접근을 검사한다. |
| `check_packet_access()` | 패킷 데이터 접근 범위를 검사한다. |

### 3. 원하는 Hook에 연결

`tc.c` 코드 내의 함수

```c
// 적재될 프로그램을 지정하는 FD 반환
tc_opts.prog_fd = bpf_program__fd(skel->progs.tc_ingress);

// // Hook과 eBPF 프로그램 연결
err = bpf_tc_attach(&tc_hook, &tc_opts);
```

TC Hook 코드 - `net/core/dev.c`

```c
sch_handle_ingress(struct sk_buff *skb, struct packet_type **pt_prev, int *ret,
		   struct net_device *orig_dev, bool *another)
{
	struct bpf_mprog_entry *entry = rcu_dereference_bh(skb->dev->tcx_ingress);
	enum skb_drop_reason drop_reason = SKB_DROP_REASON_TC_INGRESS;
	struct bpf_net_context __bpf_net_ctx, *bpf_net_ctx;
	int sch_ret;

	if (!entry)
		return skb;

	bpf_net_ctx = bpf_net_ctx_set(&__bpf_net_ctx);
	if (*pt_prev) {
		*ret = deliver_skb(skb, *pt_prev, orig_dev);
		*pt_prev = NULL;
	}

	qdisc_skb_cb(skb)->pkt_len = skb->len;
	tcx_set_ingress(skb, true);

	if (static_branch_unlikely(&tcx_needed_key)) {
		sch_ret = tcx_run(entry, skb, true);
		if (sch_ret != TC_ACT_UNSPEC)
			goto ingress_verdict;
	}
	sch_ret = tc_run(tcx_entry(entry), skb, &drop_reason);
```

- 연결된 eBPF 실행 - `net/sched/cls_bpf.c`

```c
TC_INDIRECT_SCOPE int cls_bpf_classify(struct sk_buff *skb,
				       const struct tcf_proto *tp,
				       struct tcf_result *res)
{
	struct cls_bpf_head *head = rcu_dereference_bh(tp->root);
	bool at_ingress = skb_at_tc_ingress(skb);
	struct cls_bpf_prog *prog;
	int ret = -1;

	list_for_each_entry_rcu(prog, &head->plist, link) {
		int filter_res;

		qdisc_skb_cb(skb)->tc_classid = prog->res.classid;

		if (tc_skip_sw(prog->gen_flags)) {
			filter_res = prog->exts_integrated ? TC_ACT_UNSPEC : 0;
		} else if (at_ingress) {
			/* It is safe to push/pull even if skb_shared() */
			__skb_push(skb, skb->mac_len);
			bpf_compute_data_pointers(skb); // 패킷 데이터 포인터
			filter_res = bpf_prog_run(prog->filter, skb);
			/*
			prog -> filter: 연결되어 있는 BPF 프로그램
			skb: 현재 처리하는 패킷
			bpf_prog_run(): 해당 패킷을 대상으로 BPF 프로그램을 실행
			*/
			__skb_pull(skb, skb->mac_len);
		} else {
			bpf_compute_data_pointers(skb);
			filter_res = bpf_prog_run(prog->filter, skb);
		}
		if (unlikely(!skb->tstamp && skb->tstamp_type))
			skb->tstamp_type = SKB_CLOCK_REALTIME;

		if (prog->exts_integrated) {
			res->class   = 0;
			res->classid = TC_H_MAJ(prog->res.classid) |
				       qdisc_skb_cb(skb)->tc_classid;

...
```

**Cilium에서는 Cilium Agent가 eBPF 프로그램의 적재와 연결을 관리**

### 4. eBPF 프로그램 실행

```c
// SPDX-License-Identifier: (LGPL-2.1 OR BSD-2-Clause)
/* Copyright (c) 2022 Hengqi Chen */
#include <vmlinux.h>
#include <bpf/bpf_endian.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>

#define TC_ACT_OK 0 /* TC의 해당 처리에서 패킷을 허용하고 처리 진행*/
#define ETH_P_IP  0x0800 /* Internet Protocol packet	*/

SEC("tc")
int tc_ingress(struct __sk_buff *ctx)
{
	void *data_end = (void *)(__u64)ctx->data_end; // 패킷 데이터 끝
	void *data = (void *)(__u64)ctx->data; // 패킷 데이터 시작
	struct ethhdr *l2; // Ethernet 헤더 포인터
	struct iphdr *l3; // IPv4 헤더 포인터

	if (ctx->protocol != bpf_htons(ETH_P_IP)) // IPv4가 아니면 바로 통과
		return TC_ACT_OK;

	l2 = data;
	if ((void *)(l2 + 1) > data_end) // Ethernet 헤더를 온전히 읽을 수 없으면 통과
		return TC_ACT_OK;

	l3 = (struct iphdr *)(l2 + 1);
	if ((void *)(l3 + 1) > data_end) // IPv4 기본 헤더를 온전히 읽을 수 없으면 통과
		return TC_ACT_OK;

	bpf_printk("Got IP packet: tot_len: %d, ttl: %d", bpf_ntohs(l3->tot_len), l3->ttl); // 확인한 IPv4 헤더 정보를 출력
	return TC_ACT_OK; // TC_ACT_SHOT 이면 패킷을 차단하고 버림
}

char __license[] SEC("license") = "GPL"; // 라이선스 정보를 커널에 전달
```

```c
// net/core/dev.c
case TC_ACT_SHOT: // TC ingress 차단
    kfree_skb_reason(skb, drop_reason);
```

---

- 참고자료
    
    https://github.com/libbpf/libbpf-bootstrap/blob/master/examples/c/tc.bpf.c
    
    https://github.com/libbpf/libbpf-bootstrap/blob/master/examples/c/tc.c
    
    https://github.com/torvalds/linux/blob/v6.12/net/core/dev.c#L4059
    
    https://github.com/torvalds/linux/blob/v6.12/net/sched/cls_bpf.c#L81
