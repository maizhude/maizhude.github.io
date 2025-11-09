---
layout:     post
title:     TCCS实现之修改ryu和ovs源码实现自定义action
date:       2023-3-25
author:     MZ
header-img: img/post-bg-debug.png
catalog: true
categories:
    - NS3
tags:
    - SDN
    - Open vSwitch
    - OpenFlow
---

## 一、RYU源码修改之自定义action

> 原文链接：
>
> [https://blog.csdn.net/qq_43519779/article/details/117172000]: 
>
> 

<!--more-->

1.打开 ryu/ofproto/ofproto_v1_3.py 文件，在 255 行 OFPAT_POP_PBB 后面添加上文中在 OVS 中自定义的 action 及其 action code。

```python
...
OFPAT_POP_PBB = 27              # Pop the outer PBB service tag (I-TAG)
OFPAT_SET_RWND = 30
OFPAT_EXPERIMENTER = 0xffff
...
```

2.在 280 行后面添加以下代码。

```python
# struct ofp_action_group
OFP_ACTION_GROUP_PACK_STR = '!HHI'
OFP_ACTION_GROUP_SIZE = 8
assert calcsize(OFP_ACTION_GROUP_PACK_STR) == OFP_ACTION_GROUP_SIZE

# struct ofp_action_set_rwnd
OFP_ACTION_SET_RWND_PACK_STR = '!HHH2x'
OFP_ACTION_SET_RWND_SIZE = 8
assert calcsize(OFP_ACTION_SET_RWND_PACK_STR) == OFP_ACTION_SET_RWND_SIZE
```

**注意：总大小必须是 8 的整数倍，不足需要用 nx 填充，并且还需要注意填充位置，否则会出错。例如：action 的参数为 uint64_t 时，OFP_ACTION_SET_RWND_PACK_STR 应为’!HH4xQ’。**

 简单解释：OFP_ACTION_SET_RWND_PACK_STR 表示格式化字符串，即告诉 RYU 如何将内存中的字节数据与 action 中的变量对应起来。各位还记得在给 OVS 添加自定义 aciton 的时候，我们所定义的接收参数所用结构体吧，前面两个 H 与 struct ofpact ofpact; 对应，后面的H表示 uint16_t rwnd;

> 注意x的填充位置，一开始我写成!HH2xH，发现获取不到数据就改正确了，自己不懂字节顺序的可以试一试，也可以看下面的表格

​	*字节顺序*

| 字符 | 字节顺序 |
| :--: | :------: |
|  @   |   本机   |
|  =   |   本机   |
|  <   |   小端   |
|  >   |   大端   |
|  !   |   网络   |

*数据格式*

| 字符 |      数据类型      | 字节数 |
| :--: | :----------------: | :----: |
|  nx  |        填充        |   n    |
|  c   |        char        |   1    |
|  b   |    signed char     |   1    |
|  B   |   unsigned char    |   1    |
|  ?   |        bool        |   1    |
|  h   |       short        |   2    |
|  H   |   unsigned short   |   2    |
|  i   |        int         |   4    |
|  I   |    unsigned int    |   4    |
|  l   |        long        |   4    |
|  L   |   unsigned long    |   4    |
|  q   |     long long      |   8    |
|  Q   | unsigned long long |   8    |
|  f   |       float        |   4    |
|  d   |       double       |   8    |

3.打开 ryu/ofproto/ofproto_v1_3_parser.py 文件，在 3017 行 class OFPAction 后面添加自定义 action 所对应的解析和封装类。

```python
@OFPAction.register_action_type(ofproto.OFPAT_SET_RWND,
                                ofproto.OFP_ACTION_SET_RWND_SIZE)
class OFPActionSetRWND(OFPAction):
    """
    Test action

    This action indicates test something.

    ================ ======================================================
    Attribute        Description
    ================ ======================================================
    rwnd             uint16_t
    ================ ======================================================
    """
    def __init__(self, rwnd, type_=None, len_=None):
        super(OFPActionSetRWND, self).__init__()
        self.rwnd = rwnd
    @classmethod
    def parser(cls, buf, offset):
        (type_, len_, rwnd) = struct.unpack_from(
            ofproto.OFP_ACTION_SET_RWND_PACK_STR, buf, offset)
        return cls(rwnd)

    def serialize(self, buf, offset):
        msg_pack_into(ofproto.OFP_ACTION_SET_RWND_PACK_STR, buf,
                      offset, self.type, self.len, self.rwnd)
```

### 验证自定义 Action

```python
...
actions = [parser.OFPActionSetRWND(512), parser.OFPActionOutput(out_port)]
...
```

动作集中加入该动作就好

重新编译并运行，接下来就是ovs接收该值并显示出来.

## 二、ovs源码修改之自定义action

> 前面部分参考某乎文章：https://zhuanlan.zhihu.com/p/27802666

### **1.定义openflow action**

所有action定义在lib/ofp-actions.c

```c
enum ofp_raw_action_type {
	/* ... */

	/* NX1.3+(47): struct nx_action_decap, ... */
	NXAST_RAW_DECAP,

+	/* OF1.0+(30): uint16_t. */
+    OFPAT_RAW_SET_RWND,

	/* ... */
}
```

### **2.定义openvswitch action**

打开 include/openvswitch/ofp-actions.h 文件，在 142 行，GOTO_TABLE 后面添加自己的 action 声明。

```c
#define OFPACTS
	/* ... */
    OFPACT(GOTO_TABLE,      ofpact_goto_table,  ofpact, "goto_table") \
+    OFPACT(SET_RWND,        ofpact_set_rwnd,    ofpact, "set_rwnd")
```

 在 1105 行的结构体 struct ofpact_decap 后面为自己的 action 添加接收参数所用结构体。

```c
/* ..., after "struct ofpact_decap { ... }" */

struct ofpact_set_rwnd {
    OFPACT_PADDED_MEMBERS(
        struct ofpact ofpact;
        /* 注意，这里不能使用 char 数组或指针作为变量 */
        uint16_t rwnd;
    );
    // uint8_t data[];
};
```

分别在 505、8078、8990 行后面添加自己的 action，其作用可参考文件中的函数声明。

```c
/* Returns the ofpact following 'ofpact', except that if 'ofpact' contains
 * nested ofpacts it returns the first one. */
struct ofpact *
ofpact_next_flattened(const struct ofpact *ofpact)
{
	switch (ofpact->type) {
		/* ... */
+		case OFPACT_SET_RWND:
			return ofpact_next(ofpact);
	}
	/* ... */
}

/* ... */

enum ovs_instruction_type
ovs_instruction_type_from_ofpact_type(enum ofpact_type type,
                                      enum ofp_version version)
{
	switch (type) {
	/* ... */
+	case OFPACT_SET_RWND:
	default:
		return OVSINST_OFPIT11_APPLY_ACTIONS;
	/* ... */
	}
}

/* ... */

/* Returns true if 'action' outputs to 'port', false otherwise. */
static bool
ofpact_outputs_to_port(const struct ofpact *ofpact, ofp_port_t port)
{
	switch (ofpact->type) {
	/* ... */
+	case OFPACT_SET_RWND:
	default:
		return false;
	}
}
```

在 ofp-action.c的7779 行后面为自己的 action 添加宏定义。

```c
/* The order in which actions in an action set get executed.  This is only for
 * the actions where only the last instance added is used. */
#define ACTION_SET_ORDER                        \
		/* ... */
    SLOT(OFPACT_DEC_NSH_TTL)					\
+    SLOT(OFPACT_SET_RWND)  
```

在 8788 行，get_ofpact_map 方法里分别为各个 openflow 版本添加自定义的 action。

```c
static const struct ofpact_map *
get_ofpact_map(enum ofp_version version)
{
    /* OpenFlow 1.0 actions. */
    static const struct ofpact_map of10[] = {
		/* ... */
+        { OFPACT_SET_RWND, 30},
        { 0, -1 },
    };
	
    /* OpenFlow 1.1 actions. */
    static const struct ofpact_map of11[] = {
		/* ... */
+        { OFPACT_SET_RWND, 30},
        { 0, -1 },
    };
    
    /* OpenFlow 1.2, 1.3, and 1.4 actions. */
    static const struct ofpact_map of12[] = {
		/* ... */
+        { OFPACT_SET_RWND, 30},
        { 0, -1 },
    };
	/* ... */
}
```

在 7637 行，GOTO_TABLE 后面为自己的 action 添加命令行以流表参数解析，格式化。

```c
/* Set-RWND instruction. */
static void
encode_SET_RWND(const struct ofpact_set_rwnd *setRwnd,
            enum ofp_version ofp_version OVS_UNUSED,
            struct ofpbuf *out)
{
    put_OFPAT_SET_RWND(out, setRwnd->rwnd);
}

static enum ofperr
decode_OFPAT_RAW_SET_RWND(uint16_t rwnd,
                        enum ofp_version ofp_version OVS_UNUSED,
                        struct ofpbuf *out)
{
    ofpact_put_SET_RWND(out)->rwnd = rwnd;
    return 0;
}

static char * OVS_WARN_UNUSED_RESULT
parse_SET_RWND(char *arg, const struct ofpact_parse_params *pp)
{
    uint16_t rwnd;
    char *error;
    struct ofpact_set_rwnd *setRwnd;

    error = str_to_u16(arg, "cwnnd", &rwnd);
    if (error) return error;

    setRwnd = ofpact_put_SET_RWND(pp->ofpacts);
    setRwnd->rwnd = rwnd;

    return NULL;
}


static void
format_SET_RWND(const struct ofpact_set_rwnd *a,
                const struct ofpact_format_params *fp)
{
    /* Feel free to use e.g. colors.param,
    colors.end around parameter names */
    ds_put_format(fp->s, "%sset_rwnd:%"PRIu32"%s", colors.param, a->rwnd, colors.end);
}

static enum ofperr
check_SET_RWND(const struct ofpact_set_rwnd *a OVS_UNUSED,
                const struct ofpact_check_params *cp OVS_UNUSED)
{
    /* My method needs no checking. Probably. */
    return 0;
}
```

 打开 ofproto/ofproto-dpif-xlate.c，分别在 5687、5988、6649 行后面添加自己的 action，其作用可参考文件中的函数声明。

```c
/* Determine if an datapath action translated from the openflow action
 * can be reversed by another datapath action.
 *
 * Openflow actions that do not emit datapath actions are trivially
 * reversible. Reversiblity of other actions depends on nature of
 * action and their translation.  */
static bool
reversible_actions(const struct ofpact *ofpacts, size_t ofpacts_len)
{
	const struct ofpact *a;

	OFPACT_FOR_EACH (a, ofpacts, ofpacts_len) {
		switch (a->type) {
		/*... */
+		case OFPACT_SET_RWND:
			return false;
		}
	}
	return true;
}

/* ... */

/* Copy actions 'a' through 'end' to ctx->frozen_actions, which will be
 * executed after thawing.  Inserts an UNROLL_XLATE action, if none is already
 * present, before any action that may depend on the current table ID or flow
 * cookie. */
static void
freeze_unroll_actions(const struct ofpact *a, const struct ofpact *end,
                      struct xlate_ctx *ctx)
{
	/* ... */
	switch (a->type) {
+		case OFPACT_SET_RWND:
			/* These may not generate PACKET INs. */
			break;
	}
}

/* ... */

static void
recirc_for_mpls(const struct ofpact *a, struct xlate_ctx *ctx)
{
	/* ... */
	switch (a->type) {
+	case OFPACT_SET_RWND:
	default:
		break;
	}
}
```

到此为止，自定义的 action 已经添加进 OVS 了，但我们还没有去实现 action，接下来就是最后一步，实现 action。

到这为止基本上都是按照别人的代码简单改一改就完成的，接下来的工作是实现自己的action逻辑，所以必须自己写，我的代码逻辑是**修改交换机接收数据包的TCP接收窗口大小。**

### **3.openflow action 实现**

首先，修改flow结构体，找到flow结构体发现没有rwnd的字段，于是自定义一个tp_window的字段，但要保证每64位一分片，多余用pad补齐。

include/openvswitch/flow.h

```c++
struct flow {
    /* Metadata */
   	/*....*/
    /* L4 (64-bit aligned) */
    ovs_be16 tp_src;            /* TCP/UDP/SCTP source port/ICMP type. */
    ovs_be16 tp_dst;            /* TCP/UDP/SCTP destination port/ICMP code. */
+    ovs_be16 tp_window;           /* TCP Windows size. */
+    ovs_be16 pad3;              /* Pad to 64 bits. */
    ovs_be16 ct_tp_src;         /* CT original tuple source port/ICMP type. */
    ovs_be16 ct_tp_dst;         /* CT original tuple dst port/ICMP code. */
    ovs_be32 igmp_group_ip4;    /* IGMP group IPv4 address/ICMPv6 ND reserved
                                 * field.
                                 * Keep last for BUILD_ASSERT_DECL below. */
—    //ovs_be32 pad3;              /* Pad to 64 bits. */
};
BUILD_ASSERT_DECL(sizeof(struct flow) % sizeof(uint64_t) == 0);
BUILD_ASSERT_DECL(sizeof(struct flow_tnl) % sizeof(uint64_t) == 0);
BUILD_ASSERT_DECL(sizeof(struct ovs_key_nsh) % sizeof(uint64_t) == 0);

#define FLOW_U64S (sizeof(struct flow) / sizeof(uint64_t))

/* Remember to update FLOW_WC_SEQ when changing 'struct flow'. */
* BUILD_ASSERT_DECL(offsetof(struct flow, igmp_group_ip4)
                  == sizeof(struct flow_tnl) + sizeof(struct ovs_key_nsh) + 300
                  && FLOW_WC_SEQ == 42);
```

> tp_rwnd与pad3（ovs_be16）是新添加的字段，屏蔽掉了原来的32位的pad3，因为要64位分片，所以要这样设计。

改完flow结构体，断言报错，看了一下是因为判断大小的问题，之后编译运行发现报错的地方有的需要修改。

然后实现动作修改flow的tp_window字段，在do_xlate_actions方法下添加代码：

```c++
static void
do_xlate_actions(const struct ofpact *ofpacts, size_t ofpacts_len,
                 struct xlate_ctx *ctx, bool is_last_action,
                 bool group_bucket_action)
{
     switch (a->type) {
+        case OFPACT_SET_RWND:{
+            flow->tp_window = ofpact_get_SET_RWND(a)->rwnd;
+            break;
        }
     }
}
```

> 其实这里的修改是伪修改，修改之后ovs内核并没有修改数据包相应的数据，所以要把得到的数据传到内核中，再利用内核的方法修改数据包。

修改之后commit提交到内核中：

```c++
static void
commit_set_rwnd(const struct flow* flow, struct flow *base, struct ofpbuf *odp_actions){
    struct ovs_action_set_rwnd setRwnd;
    if(flow->tp_window){
        setRwnd.rwnd = htons(flow->tp_window);
        nl_msg_put_unspec(odp_actions, OVS_ACTION_ATTR_SET_RWND, &setRwnd, sizeof setRwnd);
        base->tp_window = flow->tp_window;
    }
}
.
.
.
enum slow_path_reason
commit_odp_actions(const struct flow *flow, struct flow *base,
                   struct ofpbuf *odp_actions, struct flow_wildcards *wc,
                   bool use_masked, bool pending_encap, bool pending_decap,
                   struct ofpbuf *encap_data)
{
    //...
    commit_set_priority_action(flow, base, odp_actions, wc, use_masked);
    commit_set_pkt_mark_action(flow, base, odp_actions, wc, use_masked);
+    commit_set_rwnd(flow, base, odp_actions);
    return slow1 ? slow1 : slow2;
}
```

> ovs_action_set_rwnd以及OVS_ACTION_ATTR_SET_RWND通过编译找到需要修改的地方进行修改。

datapath/linux/compat/include/linux/openvswitch.h

```c++
enum ovs_action_attr {
#endif
+	OVS_ACTION_ATTR_SET_RWND = 30,		/* u16 rwnd size. */
	__OVS_ACTION_ATTR_MAX,	      /* Nothing past this will be accepted */
}

struct ovs_action_set_rwnd {
	__be16 rwnd;
};
```

lib/odp-util.c设置长度

```c++
static int
odp_action_len(uint16_t type)
{
+     case OVS_ACTION_ATTR_SET_RWND: return sizeof(struct ovs_action_set_rwnd);
}
```

> 其他的关于OVS_ACTION_ATTR_SET_RWND的一般很简单，要么不实现，要么就是true false break之类的，具体可以参考源码。

### **4.openvswitch内核动作实现**

首先直接进入到实现的action.c文件当中：

```c++
/* Execute a list of actions against 'skb'. */
static int do_execute_actions(struct datapath *dp, struct sk_buff *skb,
			      struct sw_flow_key *key,
			      const struct nlattr *attr, int len)
{
    switch (nla_type(a)) {
            case OVS_ACTION_ATTR_SET_RWND:
				err = execute_set_rwnd(skb, key, nla_data(a));
				break;
    }
}

//execute_set_rwnd方法
static int execute_set_rwnd(struct sk_buff *skb, struct sw_flow_key *flow_key, const struct ovs_action_set_rwnd *setRwnd)
{
    printk("haha pkt len %u, with data: 0x%u\n", skb->len, ntohl(setRwnd->rwnd));
	set_tcp_rwnd(skb, flow_key, setRwnd->rwnd);
	return 0;
}

//set_tcp_rwnd方法
static int set_tcp_rwnd(struct sk_buff *skb, struct sw_flow_key *flow_key, __be16 rwnd)
{
	struct tcphdr *th;
	__be16 window;
	int err;

	err = skb_ensure_writable(skb, skb_transport_offset(skb) +
				  sizeof(struct tcphdr));
	if (unlikely(err))
		return err;

	th = tcp_hdr(skb);
	// window = OVS_MASKED(th->window, key->tcp_dst, mask->tcp_dst);
	printk("receive window data: 0x%u\n", ntohs(th->window));
	window = rwnd;
	if (likely(window != th->window)) {
		set_tp_port(skb, &th->window, window, &th->check);
		flow_key->tp.window = window;
	}
	skb_clear_hash(skb);

	return 0;
}
```

> 其实可以不需要execute_set_rwnd方法，这里我只是打印了一下，可以直接调用set_tcp_rwnd方法，然后把接收参数改一改就行。
>
> 最后一个问题就是sw_flow_key结构中没有tp.window这个字段，所以也要修改

修改key和从skb提取key的函数：

```c
//datapath/flow.h
struct sw_flow_key {
    struct {
		__be16 src;		/* TCP/UDP/SCTP source port. */
		__be16 dst;		/* TCP/UDP/SCTP destination port. */
		__be16 flags;		/* TCP flags. */
+		__be16 window;		/* TCP rwnd size. */
	} tp;
}

//datapath/flow.c
static int key_extract_l3l4(struct sk_buff *skb, struct sw_flow_key *key)
{	
    /*....*/
	/* Transport layer. */
		if (key->ip.proto == IPPROTO_TCP) {
			if (tcphdr_ok(skb)) {
				struct tcphdr *tcp = tcp_hdr(skb);
				key->tp.src = tcp->source;
				key->tp.dst = tcp->dest;
				key->tp.flags = TCP_FLAGS_BE16(tcp);
+				key->tp.window = tcp->window;
			} /*....*/
}
```

> 把skb的tcp报头提取出来设置rwnd字段。key->tp.window = tcp->window;

### 5.最终结果

在单个交换机的简单哑铃结构下，设置ACK数据包的窗口值为1024，可以从下面看出设置成功了。

![image-20230924120710547]({{ site.url }}/assets/setrwnd.png)

![image-20230924115701902]({{ site.url }}/assets/modify_rwnd.png)
