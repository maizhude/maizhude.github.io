---
layout:     post
title: 修改适应ns3的bofuss库源码之自定义NewMatchField
date:       2023-9-24
author:     MZ
header-img: img/post-bg-iWatch.jpg
catalog: true
categories:
    - NS3
tags:
    - ns3
    - SDN
    - OpenFlow
---

# 新增匹配字段——支持TCP标志位

## 一、定义匹配字段

> 由于版本更新，[Wiki](https://github.com/CPqD/ofsoftswitch13/wiki/Adding-a-New-Match-Field)的教程使用NetPDL修改解析数据包的方法不适用，本文结合教程以及[论文](https://dl.acm.org/doi/pdf/10.1145/3532577.3532608)的实现过程给出以下修改步骤。本案例新增的匹配字段是TCP标志位。<!--more-->

### 1.定义结构体并解析

openflow.h

```c
enum oxm_ofb_match_fields {
    OFPXMT_OFB_IN_PORT        = 0,  /* Switch input port. */
    ...
    OFPXMT_OFB_TCP_FLAGS = 42, /* TCP flags. */
};
/* The flags in the TCP header.
*
* Prereqs:
* OXM_OF_ETH_TYPE must be either 0x0800 or 0x86dd.
* OXM_OF_IP_PROTO must match 6 exactly.
*
* Format: 16-bit integer with 4 most-significant bits forced to 0.
*
* Masking: Bits 0-11 fully maskable. */
#define OXM_OF_TCP_FLAGS OXM_HEADER (0x8000, OFPXMT_OFB_TCP_FLAGS, 2)
#define OXM_OF_TCP_FLAGS_W OXM_HEADER_W(0x8000, OFPXMT_OFB_TCP_FLAGS, 2)
```

oflib/oxm-match.def

```def
DEFINE_FIELD_M  (OF_TCP_FLAGS,  OXM_DL_IP_ANY,   IPPROTO_TCP,    true)
```

oflib/oxm-match.h

```c
enum oxm_field_index {
#define DEFINE_FIELD(HEADER,DL_TYPES, NW_PROTO, MASKABLE) \
        OFI_OXM_##HEADER,
#include "oxm-match.def"
    NUM_OXM_FIELDS = 58
};
```

oflib/oxm-match.c

```c
case OFI_OXM_OF_TCP_FLAGS:{
    uint16_t* tcp_flags = (uint16_t*) value;
    ofl_structs_match_put16(match, f->header, ntohs(*tcp_flags));
    return 0;
}

case OFI_OXM_OF_TCP_FLAGS_W:{
     uint16_t tcp_flags = ntohs((uint16_t*) value));
     uint16_t tcp_flags_mask = ntohs(*((uint16_t*) mask));

     if (check_bad_wildcard16(tcp_flags, tcp_flags_mask)){
         return ofp_mkerr(OFPET_BAD_MATCH, OFPBMC_BAD_WILDCARDS);
     }
     /* According to OpenFlow only the bits, from the least significant bit, 0 to 11 are valid
        while the remaining should be always set to 0.
      */
     //if (tcp_flags & F000){
     //  return ofp_mkerr(OFPET_BAD_MATCH, OFPBMC_BAD_VALUE);
     //}
  
     ofl_structs_match_put16m(match, f->header, tcp_flags, tcp_flags_mask);
}
```

### 2.控制台打印信息

ofl_struct_print.c

```c
#define TCP_FIN 0x01
#define TCP_SYN 0x02
#define TCP_RST 0x04
#define TCP_PSH 0x08
#define TCP_ACK 0x10
#define TCP_URG 0x20

case OFPXMT_OFB_TCP_FLAGS:{
    uint16_t value = *((uint16_t*) f->value);
    char str[64] = "";
    if (value & TCP_FIN) {
        strcat(str, "FIN ");
    }
    if (value & TCP_SYN) {
        strcat(str, "SYN ");
    }
    if (value & TCP_RST) {
        strcat(str, "RST ");
    }
    if (value & TCP_PSH) {
        strcat(str, "PSH ");
    }
    if (value & TCP_ACK) {
        strcat(str, "ACK ");
    }
    if (value & TCP_URG) {
        strcat(str, "URG ");
    }
    fprintf(stream, "tcp_flags=\"%s\"", str);
    break;
}
```

打印结果展示：

```shell
1s [dp_acts|DBG] action result: pkt{in="1", actset=[], pktout="0", ogrp="any", oprt="flood", buffer="none", ns3pktid="3", changes="0", clone="0", std={proto={eth,ipv4,tcp}, match=oxm{in_port="1", metadata="0x0", eth_dst="00:00:00:00:00:03", eth_src="00:00:00:00:00:01", eth_type="0x800", ip_dscp="0", ip_ecn="0", ipv4_src="10.0.0.1", ipv4_dst="10.0.0.2", ip_proto="6", tcp_src="49153", tcp_dst="5000", tunnel_id="0x0", tcp_flags="SYN "}"}}
```

## 二、数据包解析时增加新Match字段的解析

> packet_handle_std.c中新增一行代码
>
> ofl_structs_match_put16 (m, OXM_OF_TCP_FLAGS, ntohs (proto->tcp->tcp_ctl));

```c
/* TCP */
if (next_proto == IP_TYPE_TCP) {
    if (unlikely (pkt->buffer->size < offset + sizeof (struct tcp_header))) return;
    proto->tcp = (struct tcp_header *)((uint8_t *)pkt->buffer->data + offset);
    offset += sizeof (struct tcp_header);

    ofl_structs_match_put16 (m, OXM_OF_TCP_SRC, ntohs (proto->tcp->tcp_src));
    ofl_structs_match_put16 (m, OXM_OF_TCP_DST, ntohs (proto->tcp->tcp_dst));
    ofl_structs_match_put16 (m, OXM_OF_TCP_FLAGS, ntohs (proto->tcp->tcp_ctl));

    /* No processing past TCP */
    return;
}
```

flow_table.c

```c
uint32_t  oxm_ids[]={OXM_OF_IN_PORT,..., OXM_OF_TCP_FLAGS};
```

## 三、流表与数据包匹配时增加新Match的匹配规则

> 从pipeline处理数据包找到流表与数据包相匹配的函数，新增有关flag的代码，这里的flag虽然长度为2，但是值只有8位，所以只解析8位就行，16位会解析错误。
>
> ![查找函数路径]({{	site.url	}}/assets/flow.png)

```c++
bool
packet_match(struct ofl_match *flow_match, struct ofl_match *packet){

    
	//...
    /* Loop over the flow entry's match fields */
    HMAP_FOR_EACH(f, struct ofl_match_tlv, hmap_node, &flow_match->match_fields)
    {
        /* Compare the flow and packet field values, considering the mask, if any */
        packet_val = packet_f->value;
        switch (field_len) {
            //...
            case 2:
                switch (packet_header) {
                	//...
                    case OXM_OF_TCP_FLAGS: {
                        if (has_mask) {
                            if (!match_mask8(flow_val, flow_mask, packet_val))
                                return false;
                        }
                        else {
                            if (!match_8(flow_val, packet_val))
                                return false;
                        }
                        break;
                    }
                    default:
                       //...
                }
                break;
            case 3:
                //...
            case 4:
                //...
            case 6:
                //...
            case 8:
                //...
            case 16:
                //...
            default:
                /* Should never happen */
                break;
        }
    }
    /* If we get here, all match fields in the flow entry matched the packet */
    return true;
}
```



## 四、工具dpctl支持新Match字段

dpctl.h

```c
#define MATCH_TP_FLAGS       "tcp_flags"
```

dpctl.c 在tcp_dst后面增加类似函数解析

```c
if (strncmp(token, MATCH_TP_FLAGS KEY_VAL, strlen(MATCH_TP_FLAGS KEY_VAL)) == 0) {
            uint16_t tcp_flags;
            if (parse16(token + strlen(MATCH_TP_FLAGS KEY_VAL), NULL, 0, 0xffff, &tcp_flags)) {
                ofp_fatal(0, "Error parsing tcp_flags: %s.", token);
            }
            else ofl_structs_match_put16(m, OXM_OF_TCP_FLAGS,tcp_flags);
            continue;
        }
```

## 五、测试

> 新增Match Field成功，在Controller接受测试，在packet_in中加入下面这段代码

```c++
uint16_t isTCP;
struct ofl_match_tlv *ip_proto =
    oxm_match_lookup (OXM_OF_IP_PROTO, (struct ofl_match*)msg->match);
if(ip_proto != NULL){
    memcpy(&isTCP, ip_proto->value, OXM_LENGTH(OXM_OF_IP_PROTO));
    if(isTCP == 6){
        Ipv4Address ipv4_src;
        struct ofl_match_tlv *ipv4Src =
            oxm_match_lookup (OXM_OF_IPV4_SRC, (struct ofl_match*)msg->match);
        memcpy(&ipv4_src, ipv4Src->value, OXM_LENGTH(OXM_OF_IPV4_SRC));

        Ipv4Address ipv4_dst;
        struct ofl_match_tlv *ipv4Dst =
            oxm_match_lookup (OXM_OF_IPV4_DST, (struct ofl_match*)msg->match);
        memcpy(&ipv4_dst, ipv4Dst->value, OXM_LENGTH(OXM_OF_IPV4_DST));

        struct ofl_match_tlv* tlv;
        uint16_t srcPort;
        uint16_t dstPort;
        tlv = oxm_match_lookup(OXM_OF_TCP_SRC, (struct ofl_match*)msg->match);
        memcpy(&srcPort, tlv->value, OXM_LENGTH(OXM_OF_TCP_SRC));
        tlv = oxm_match_lookup(OXM_OF_TCP_DST, (struct ofl_match*)msg->match);
        memcpy(&dstPort, tlv->value, OXM_LENGTH(OXM_OF_TCP_DST));
        int tcpFlags;
        tlv = oxm_match_lookup(OXM_OF_TCP_FLAGS, (struct ofl_match*)msg->match);
        memcpy(&tcpFlags, tlv->value, OXM_LENGTH(OXM_OF_TCP_FLAGS));
        if (tcpFlags & TCP_FIN) {
            NS_LOG_DEBUG ("TCP FLAG IS: TCP_FIN");
        }
        if (tcpFlags & TCP_SYN) {
            NS_LOG_DEBUG ("TCP FLAG IS: TCP_SYN");
        }
        if (tcpFlags & TCP_RST) {
            NS_LOG_DEBUG ("TCP FLAG IS: TCP_RST");
        }
        if (tcpFlags & TCP_PSH) {
            NS_LOG_DEBUG ("TCP FLAG IS: TCP_PSH");
        }
        if (tcpFlags & TCP_ACK) {
            NS_LOG_DEBUG ("TCP FLAG IS: TCP_ACK");
        }
        if (tcpFlags & TCP_URG) {
            NS_LOG_DEBUG ("TCP FLAG IS: TCP_URG");
        }
        NS_LOG_DEBUG ("-----------------------");

    }
}
```

> HandshakeSuccessful中添加下发流表的代码，这样遇到SYN数据包就能发送到控制器

```c++
DpctlExecute (swDpId, "flow-mod cmd=add,table=0,prio=300 eth_type=0x800,ip_proto=6,tcp_flags=2 apply:output=ctrl:128");
```

> 结果

```sh
OFSwitch13LearningController:OFSwitch13LearningController(0x56191b380080)
OFSwitch13LearningController:HandshakeSuccessful(0x56191b380080, 0x56191b24a350)
OFSwitch13LearningController:HandlePacketIn(0x56191b380080, 0x56191b24a350, 0)
Packet in match: oxm{in_port="1", metadata="0x0", eth_dst="ff:ff:ff:ff:ff:ff", eth_src="00:00:00:00:00:01", eth_type="0x806", arp_op="0x1", arp_spa="10.0.0.1", arp_tpa="10.0.0.2", arp_sha="00:00:00:00:00:01", arp_tha="ff:ff:ff:ff:ff:ff", tunnel_id="0x0"}
Learning that mac 00:00:00:00:00:01 can be found at port 1
OFSwitch13LearningController:HandlePacketIn(0x56191b380080, 0x56191b24a350, 0)
Packet in match: oxm{in_port="1", metadata="0x0", eth_dst="00:00:00:00:00:03", eth_src="00:00:00:00:00:01", eth_type="0x800", ip_dscp="0", ip_ecn="0", ipv4_src="10.0.0.1", ipv4_dst="10.0.0.2", ip_proto="6", tcp_src="49153", tcp_dst="5000", tunnel_id="0x0", tcp_flags="SYN "}
TCP FLAG IS: TCP_SYN
-----------------------
No L2 info for mac 00:00:00:00:00:03. Flood.
OFSwitch13LearningController:HandlePacketIn(0x56191b380080, 0x56191b24a350, 0)
Packet in match: oxm{in_port="2", metadata="0x0", eth_dst="ff:ff:ff:ff:ff:ff", eth_src="00:00:00:00:00:03", eth_type="0x806", arp_op="0x1", arp_spa="10.0.0.2", arp_tpa="10.0.0.1", arp_sha="00:00:00:00:00:03", arp_tha="ff:ff:ff:ff:ff:ff", tunnel_id="0x0"}
Learning that mac 00:00:00:00:00:03 can be found at port 2
OFSwitch13LearningController:DoDispose(0x56191b380080)
OFSwitch13LearningController:~OFSwitch13LearningController(0x56191b380080)

```

