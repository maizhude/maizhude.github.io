---
layout:     post
title: 修改适应ns3的bofuss库源码之自定义NewMessage
subtitle:  自定义NewMessage
date:       2023-9-23
author:     MZ
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - ns3
    - SDN
---

# 自定义NewMessage

> 在这个例子中，我们新增两个消息OFPT_QUE_CN和OFPT_QUE_CR实现从交换机发送到控制器，告诉控制器目前是拥塞状态，还是恢复拥塞。使用与下面的示例类似的实验者消息来创建新动作或新消息。

## 一、定义新消息

### 1.添加两个新的类型

```c++
/* file:include/openflow/openflow.h 
 * Add OFPT_QUE_CN and OFPT_QUE_CR
 * after the last defined type
 */
enum ofp_type {
     OFPT_HELLO = 0,
     ...
     OFPT_METER_MOD = 29,
     OFPT_QUE_CN = 30,
     OFPT_QUE_CR = 31,
}
```

### 2.定义新消息的Openflow消息格式

```c++
/* file:include/openflow/openflow.h 
 * Struct representing the cn. 
 */
struct ofp_que_cn{
    struct ofp_header header;
    uint16_t queue_length;
    uint8_t pad[6];
}

/* file:include/openflow/openflow.h 
 * Struct representing the cr. 
 * All OpenFlow messages are 8 bytes 
 * aligned, thus padding is added
 */
struct ofp_que_cr{
    struct ofp_header header;
    uint16_t queue_length;
    uint8_t pad[6];
}
```

### 3.定义新消息的交换机内部数据呈现格式

```c++
/* file:oflib/ofl-messages.h
 * The common header for messages. All message structures start with this
 * header, therefore they can be safely cast back and forth */
struct ofl_msg_header {
    enum ofp_type   type;   /* One of the OFPT_ constants. */
};

/* file:oflib/ofl-messages.h
 * Internal representation of 
 * OFPT_QUE_CN and OFPT_QUE_CR.
 */ 
struct ofp_que_cn_cr{
    struct ofp_header header;
    uint16_t queue_length;
    uint8_t pad[6];
}
```

### 4.定义新消息的Openflow格式与交换机内部格式转换规则（pack与unpack）

```c++
static int
ofl_msg_pack_que_cn_cr(struct ofl_msg_que_cn_cr *msg, uint8_t **buf, size_t *buf_len) {
        struct ofp_que_cn_cr *rep;
        /* Allocates memory for the OpenFlow message
        *buf_len = sizeof(struct ofp_que_cn_cr);
        *buf     = (uint8_t *)malloc(*buf_len);
        /* Point the struct to the buffer address 
         * By doing it we can simply set the struct fields.
         */
        rep = (struct ofp_que_cn_cr *)(*buf);
        /* It is important to set the variables in network order */
        rep->queue_length =  htons(msg->queue_length);
        memset(rep->pad,0,sizeof(rep->pad));
        return 0;
}

ofl_msg_pack(struct ofl_msg_header *msg, uint32_t xid, uint8_t **buf, size_t *buf_len, struct ofl_exp *exp) {
    struct ofp_header *oh;
    int error = 0;
    switch (msg->type) {

        case OFPT_HELLO: {
            error = ofl_msg_pack_empty(msg, buf, buf_len);
            break;
        }
        /* Most messages are omited for space */
        case OFPT_ECHO_REQUEST:
        case OFPT_ECHO_REPLY: {
            error = ofl_msg_pack_echo((struct ofl_msg_echo *)msg, buf, buf_len);
            break;
        }
    }
    if (error) {
        return error;
        // TODO Zoltan: free buffer?
    }

    oh = (struct ofp_header *)(*buf);
    oh->version =        OFP_VERSION;
    oh->type    =        msg->type;
    oh->length  = htons(*buf_len);
    oh->xid     = htonl(xid);

    return 0;
}
```

```c++
static ofl_err
ofl_msg_unpack_que_cn_cr(struct ofp_header *src, size_t *len, struct ofl_msg_header **msg) {

    struct ofp_que_cn_cr *rep;
    struct ofl_que_cn_cr *irep;

    /* Check if the message has the expected size */
    if (*len < sizeof(struct  ofp_que_cn_cr)){
        OFL_LOG_WARN(LOG_MODULE, "Received MAX_QUEUE_REQUEST message has invalid length (%zu).", *len);
        return ofl_error(OFPET_BAD_REQUEST, OFPBRC_BAD_LEN);
    }
    *len -= sizeof(struct ofp_que_cn_cr);

    /* Extract the field from the OF message */
    rep = (struct ofp_que_cn_cr *) src;
    irep = (struct ofl_que_cn_cr *) malloc(sizeof(struct ofl_que_cn_cr));

    irep->queue_length = ntohs(rep->queue_length);
    *msg = (struct ofl_msg_header *)irep;
    return 0;
}

ofl_err
ofl_msg_unpack(uint8_t *buf, size_t buf_len, struct ofl_msg_header **msg, uint32_t *xid, struct ofl_exp *exp) {
    struct ofp_header *oh;
    size_t len = buf_len;
    ofl_err error = 0;
    if (len < sizeof(struct ofp_header)) {
        OFL_LOG_WARN(LOG_MODULE, "Received message is shorter than ofp_header.");
        if (xid != NULL) {
            *xid = 0x00000000;
        }
        return ofl_error(OFPET_BAD_REQUEST, OFPBRC_BAD_LEN);
    }

    oh = (struct ofp_header *)buf;

    if (oh->version != OFP_VERSION) {
        OFL_LOG_WARN(LOG_MODULE, "Received message has wrong version.");
        return ofl_error(OFPET_HELLO_FAILED, OFPHFC_INCOMPATIBLE);
    }

    if (xid != NULL) {
        *xid = ntohl(oh->xid);
    }

    if (len != ntohs(oh->length)) {
        OFL_LOG_WARN(LOG_MODULE, "Received message length does not match the length field.");
        return ofl_error(OFPET_BAD_REQUEST, OFPBRC_BAD_LEN);
    }

    switch (oh->type) {
        case OFPT_HELLO:{
            error = ofl_msg_unpack_empty(oh, &len, msg);
            break;
        }
        /* Most messages omited for space */
        case OFPT_QUE_CN:
        case OFPT_QUE_CR:
            error = ofl_msg_unpack_que_cn_cr(oh, &len, msg)
            break;
        default: {
            error = ofl_error(OFPET_BAD_REQUEST, OFPGMFC_BAD_TYPE);
        }
      ...
    (*msg)->type = (enum ofp_type)oh->type;
    return 0
}
```

## 二、在交换机端处理新消息——待完成（需要找到哪里记录队列长度，并从这里构建消息发送到控制器）

### 1.获取队列长度

因为ofswitch13只有一个OFSwitch13PriorityQueue实现，而且它继承的是Queue，Queue有一个GetNPackets()方法，返回的是The number of packets currently stored in the Queue，也就是队列里的数据包长度（当然也可以用GetNBytes()返回字节数）。

### 2.判断并发送消息到控制器

根据规则定时查询队列长度，大于某一阈值发送消息到控制器。在OFSwitch13Device中有一个SendToController方法，但是它发送的好像是数据包格式，现在不知道需不需要把自己想要传输的数据转换成数据包格式。
