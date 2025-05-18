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
    - OpenFlow
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
 * Struct representing the cn and cr. 
 */
struct ofp_msg_que_cn_cr{
    struct ofp_header header;
    uint16_t queue_length;
    uint32_t port_no;
    uint8_t pad[2];
};
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
struct ofl_msg_que_cn_cr{
    struct ofl_msg_header header;
    uint16_t queue_length;
    uint32_t port_no;
    uint8_t pad[2];
};
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
        rep->port_no =  htons(msg->port_no);
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
    irep->port_no = ntohs(rep->port_no);
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



## 二、在交换机端获取队列构建新消息

### 1.交换机获取队列长度

> 因为ofswitch13只有一个OFSwitch13PriorityQueue实现，而且它继承的是Queue，Queue有一个GetNPackets()方法，返回的是The number of packets currently stored in the Queue，也就是队列里的数据包长度（当然也可以用GetNBytes()返回字节数）。

```C++
/**
 * @brief 定时查询交换机队列长度
 * 
 * @param openFlowDev OFSwitch13Device
 */
void QueryAllQueLength(Ptr<OFSwitch13Device> openFlowDev) {
  //获取交换机的端口数量
  size_t portSize = openFlowDev->GetSwitchPortSize();
  uint64_t dpid = openFlowDev->GetDpId();
  for(uint32_t i = 0; i < portSize; i++){
    Ptr<OFSwitch13Port> ofPort = openFlowDev->GetSwitchPort(i+1);
    Ptr<OFSwitch13Queue> ofQue = ofPort->GetPortQueue();
    uint16_t queueLength = ofQue->GetNPackets();
    uint16_t state = ofPort->GetCongestionState();
    uint16_t count = ofPort->GetCongestionRecoverCount();
    uint32_t port_no = i+1;
    //判断是否大于阈值
    if(queueLength > 20){
      // NS_LOG_INFO("The Port " << i+1 << " queueLength is " << queueLength);
      if(state == 0){
        state = 1;
        ofPort->SetCongestionState(state);
      }
      openFlowDev->SendQueueCongestionNotifyMessage(dpid,queueLength,port_no);
      count = 0;
      ofPort->SetCongestionRecoverCount(count);
    }else{
      if(state == 1){
        count++;
        ofPort->SetCongestionRecoverCount(count);
      }
      if(count == 3){
        openFlowDev->SendQueueCongestionRecoverMessage(dpid,queueLength,port_no);
        // NS_LOG_INFO("The count is " << count);
        count = 0;
        state = 0;
        ofPort->SetCongestionRecoverCount(count);
        ofPort->SetCongestionState(state);
      }
    }
    //OFSwitch13Device构造发送函数，发送到控制器
  }
  
  // Reschedule the function call
  Time delay = MicroSeconds(10); // Set the desired time interval
  Simulator::Schedule(delay, &QueryAllQueLength, openFlowDev);
}
```

### 2.交换机构建新消息

> 在OFSwitch13Device（相当于一个交换机设备）中，定义SendQueueCongestionNotifyMessage函数用来构建openflow消息包括消息头和消息体
>
> ```c++
>  struct ofl_msg_que_cn_cr msg;
>   msg.header.type = OFPT_QUE_CN;
>   msg.queue_length = queueLength;
>   msg.port_no = port_no;
> ```
>
> 通过datapath的dp_send_message发送到控制器

```C++
int
OFSwitch13Device::SendQueueCongestionNotifyMessage (uint64_t dpid, uint16_t queueLength, uint32_t port_no)
{
  NS_LOG_FUNCTION (this << queueLength);
  Ptr<OFSwitch13Device> openFlowDev = GetDevice(dpid);
  Ptr<RemoteController> remoteCtrl = openFlowDev->GetFirstRemoteController();
  // Create the packet_in message.
  struct ofl_msg_que_cn_cr msg;
  msg.header.type = OFPT_QUE_CN;
  msg.queue_length = queueLength;
  msg.port_no = port_no;
  // struct sender senderCtrl;
  // senderCtrl.remote = remoteCtrl->m_remote;
  // senderCtrl.conn_id = 0; // TODO No support for auxiliary connections.
  // senderCtrl.xid = 0;
  return dp_send_message (m_datapath, (struct ofl_msg_header *)&msg, 0);
}
```

## 三、在控制器处理新消息

> 首先判断属于哪个消息，在HandleSwitchMsg中增加两个case：OFPT_QUE_CN和OFPT_QUE_CR

```C++
ofl_err
OFSwitch13Controller::HandleSwitchMsg (
  struct ofl_msg_header *msg, Ptr<RemoteSwitch> swtch, uint32_t xid)
{
  NS_LOG_FUNCTION (this << swtch << xid);

  // Dispatches control messages to appropriate handler functions.
  switch (msg->type)
    {
    case OFPT_HELLO:
      return HandleHello (msg, swtch, xid);

    case OFPT_PACKET_IN:
      return HandlePacketIn (
        (struct ofl_msg_packet_in*)msg, swtch, xid);
    //......
    case OFPT_QUE_CN:
      return HandleQueCn (
        (struct ofl_msg_que_cn_cr*)msg, swtch, xid);
    case OFPT_QUE_CR:
      return HandleQueCr (
        (struct ofl_msg_que_cn_cr*)msg, swtch, xid);
    //......
    case OFPT_EXPERIMENTER:
    default:
      return ofl_error (OFPET_BAD_REQUEST, OFPGMFC_BAD_TYPE);
    }
}
```

它们的函数声明在ofswitch13-controller.h与ofswitch13-tsfcc-controller.h中

```C++
/* ofswitch13-controller.h */
virtual ofl_err HandleQueCn (
    struct ofl_msg_que_cn_cr *msg, Ptr<const RemoteSwitch> swtch,
    uint32_t xid);

virtual ofl_err HandleQueCr (
    struct ofl_msg_que_cn_cr *msg, Ptr<const RemoteSwitch> swtch,
    uint32_t xid);
/* ofswitch13-controller.cc */
ofl_err
OFSwitch13Controller::HandleQueCn (
  struct ofl_msg_que_cn_cr *msg, Ptr<const RemoteSwitch> swtch,
  uint32_t xid)
{
  NS_LOG_FUNCTION (this << swtch << xid);

  ofl_msg_free ((struct ofl_msg_header*)msg, 0);
  return 0;
}

ofl_err
OFSwitch13Controller::HandleQueCr (
  struct ofl_msg_que_cn_cr *msg, Ptr<const RemoteSwitch> swtch,
  uint32_t xid)
{
  NS_LOG_FUNCTION (this << swtch << xid);

  ofl_msg_free ((struct ofl_msg_header*)msg, 0);
  return 0;
}
```

```c++
/* ofswitch13-tsfcc-controller.h */  
ofl_err HandleQueCn (
    struct ofl_msg_que_cn_cr *msg, Ptr<const RemoteSwitch> swtch,
    uint32_t xid);

  ofl_err HandleQueCr (
    struct ofl_msg_que_cn_cr *msg, Ptr<const RemoteSwitch> swtch,
    uint32_t xid);

/* ofswitch13-tsfcc-controller.cc */

/**
 * @brief 用于处理队列超过阈值接收到的OpenFlow消息，流程为：先区分象鼠流，再根据队列长度判断进行哪一种拥塞控制方案
 * 
 * @param msg OpenFlow消息
 * @param swtch 交换机
 * @param xid xid
 * @return ofl_err 错误结构体
 */
ofl_err
OFSwitch13TsfccController::HandleQueCn (
  struct ofl_msg_que_cn_cr *msg, Ptr<const RemoteSwitch> swtch,
  uint32_t xid)
{
  NS_LOG_FUNCTION (this << swtch << xid);
  //自己定义实现
  
  ofl_msg_free ((struct ofl_msg_header*)msg, 0);
  return 0;
}
/**
 * @brief 用于处理接收到队列长度恢复到阈值以下的OpenFlow消息，流程为：先区分象鼠流，再根据BDP等对象鼠流进行不同的rwnd值的增加
 * 
 * @param msg OpenFlow消息
 * @param swtch 交换机
 * @param xid xid
 * @return ofl_err 错误结构体
 */
ofl_err
OFSwitch13TsfccController::HandleQueCr (
  struct ofl_msg_que_cn_cr *msg, Ptr<const RemoteSwitch> swtch,
  uint32_t xid)
{
  NS_LOG_FUNCTION (this << swtch << xid);

  //自己定义实现
  ofl_msg_free ((struct ofl_msg_header*)msg, 0);
  return 0;
}
```

