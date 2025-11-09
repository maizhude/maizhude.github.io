---
layout:     post
title: 修改适应ns3的bofuss库源码之自定义ACTION
date:       2023-9-17
author:     MZ
header-img: img/post-bg-miui6.jpg
catalog: true
categories:
    - NS3
tags:
    - ns3
    - SDN
    - OpenFlow
---

# 自定义Action

## 一、定义新Action

> 在这个例子中，我们实现了一个OpenFlow动作修改TCP数据包的接收窗口值。使用与下面的示例类似的实验者消息来创建新动作或新消息。

### 1.定义新Action的字段类型

首先，OFPAT_SET_RWND字段应该添加到openflow.h文件的ofp_action_type枚举当中。

```c++
/* file:include/openflow/openflow.h 
 * Add OFPAT_POP_PBB 
 * after the last defined type
 */
enum ofp_action_type {
     OFPAT_OUTPUT       = 0,  /* Output to switch port. */
     ...
     OFPAT_PUSH_PBB     = 26, /* Push a new PBB service tag (I-TAG) */
     OFPAT_POP_PBB      = 27, /* Pop the outer PBB service tag (I-TAG) */
     OFPAT_SET_RWND     = 28, /* Set TCP RWND value. */
     OFPAT_EXPERIMENTER = 0xffff
}
```

### 2.定义新Action的Openflow消息格式

```c++
/* file:include/openflow/openflow.h 
 * Struct representing the set rwnd action. 
 * All OpenFlow messages are 8 bytes 
 * aligned, thus padding is added
 */
/* Action structure for OFPAT_SET_RWND. */
struct ofp_action_set_rwnd
{
    uint16_t type;                  /* OFPAT_SET_RWND. */
    uint16_t len;                   /* Length is 8. */
    uint16_t rwnd;                  /* rwnd value. */
    uint8_t pad[2];
};
```

### 3.定义新Action的交换机内部数据呈现格式

```c++
/* file:oflib/ofl-actions.h
 * Internal representation of 
 * OFPAT_SET_RWND.
 */ 
 struct ofl_action_set_rwnd {
    struct ofl_action_header   header; /* OFPAT_SET_RWND. */
    uint16_t rwnd;
};
```

### 4.定义新Action的Openflow格式与交换机内部格式转换规则（pack与unpack）

```c++
//pack
size_t
ofl_actions_pack(struct ofl_action_header *src, struct ofp_action_header *dst, uint8_t* data,  struct ofl_exp *exp) {

    dst->type = htons(src->type);
    memset(dst->pad, 0x00, 4);

    switch (src->type) {
        case OFPAT_OUTPUT: {
            struct ofl_action_output *sa = (struct ofl_action_output *)src;
            struct ofp_action_output *da = (struct ofp_action_output *)dst;

            da->len =     htons(sizeof(struct ofp_action_output));
            da->port =    htonl(sa->port);
            da->max_len = htons(sa->max_len);
            memset(da->pad, 0x00, 6);
            return sizeof(struct ofp_action_output);
        }
        /* Most actions are omited for space */
        case OFPAT_SET_RWND:{
            struct ofl_action_set_rwnd *sa = (struct ofl_action_set_rwnd *) src;
            struct ofp_action_set_rwnd *da = (struct ofp_action_set_rwnd *) dst;
            da->len =     htons(sizeof(struct ofp_action_set_rwnd));
            da->rwnd =    htons(sa->rwnd);
            memset(da->pad, 0x00, 2);
            return sizeof(struct ofp_action_set_rwnd);
        }
    };
}
```

```c++
ofl_err
ofl_actions_unpack(struct ofp_action_header *src, size_t *len, struct ofl_action_header **dst, struct ofl_exp *exp) {

    if (*len < sizeof(struct ofp_action_header)) {
        VLOG_WARN(LOG_MODULE, "Received action is too short (%zu).", *len);
        return ofl_error(OFPET_BAD_ACTION, OFPBAC_BAD_LEN);
    }

    if (*len < ntohs(src->len)) {
        VLOG_WARN(LOG_MODULE, "Received action has invalid length (set to %u, but only %zu received).", ntohs(src->len), *len);
        return ofl_error(OFPET_BAD_ACTION, OFPBAC_BAD_LEN);
    }

    if ((ntohs(src->len) % 8) != 0) {
        VLOG_WARN(LOG_MODULE, "Received action length is not a multiple of 64 bits (%u).", ntohs(src->len));
        return ofl_error(OFPET_BAD_ACTION, OFPBAC_BAD_LEN);
    }

    switch (ntohs(src->type)) {
        case OFPAT_OUTPUT: {
           //...
        }
        /* Most actions omited for space */
        case OFPAT_SET_RWND: {
            struct ofp_action_set_rwnd *sa;
            struct ofl_action_set_rwnd *da;

            if (*len < sizeof(struct ofp_action_set_rwnd)) {
                VLOG_WARN(LOG_MODULE, "Received SET_RWND action has invalid length (%zu).", *len);
                return ofl_error(OFPET_BAD_ACTION, OFPBAC_BAD_LEN);
            }

            sa = (struct ofp_action_set_rwnd *)src;

            da = (struct ofl_action_set_rwnd *)malloc(sizeof(struct ofl_action_set_rwnd));
            da->rwnd = sa->rwnd;

            *len -= sizeof(struct ofp_action_set_rwnd);
            *dst = (struct ofl_action_header *)da;
            break;
        }
    };
}
```



## 二、在交换机端处理新Action

### 1.处理新Action的函数

```c++
static void
set_rwnd(struct packet *pkt, struct ofl_action_set_nw_ttl *act) {
    packet_handle_std_validate(pkt->handle_std);
    if (pkt->handle_std->proto->tcp != NULL) {
        struct tcp_header *tcp = pkt->handle_std->proto->tcp;
        uint16_t old_rwnd = tcp->winsz;
        uint16_t new_rwnd = htons(act->rwnd);
        tcp->tcp_csum = recalc_csum16(tcp->tcp_csum, old_rwnd, new_rwnd);
        tcp->winsz = new_rwnd;
        packet_modified (pkt);
    }
    else {
        VLOG_WARN(LOG_MODULE, "Trying to execute SET_NW_TTL action on packet with no ipv4 or ipv6.");
    }
}
```

### 2.使用上述函数

```c++
dp_execute_action(struct packet *pkt,
                  struct ofl_action_header *action) {

    if (VLOG_IS_DBG_ENABLED(LOG_MODULE)) {
        char *a = ofl_action_to_string(action, pkt->dp->exp);
        VLOG_DBG(LOG_MODULE, "executing action %s.", a);
        free(a);
    }

    switch (action->type) {
        case (OFPAT_SET_FIELD): {
            set_field(pkt,(struct ofl_action_set_field*) action);
            break;
        }
        /* Most actions omited for space */
        case (OFPAT_SET_RWND):{
            set_rwnd(pkt, (struct ofl_action_set_rwnd*)action);
            break;
        }
    };
}
```

