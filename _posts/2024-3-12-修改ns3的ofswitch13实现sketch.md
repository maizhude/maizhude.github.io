---
layout:     post
title: 修ns3的ofswitch13源码之实现Sketch
subtitle:  实现Sketch
date:       2024-3-12
author:     MZ
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - ns3
    - SDN
    - OpenFlow
    - Sketch
---



# NS3模拟实现Sketch查找TOP-K

> 由于sketch相关实现采用的是P4原语，而在ns3中没有这方面的支持，所以现在采用模拟sketch的方案来实现sketch相关功能。具体实现是在交换机经过数据包时解析数据包获取四元组，然后计算hash值，并存入到自己设置的sketch数据结构中。最后ns3运行采取定期上传sketch中的数据，以供控制器区分长短流。

## 1.从openflow交换机的  pipeline中获取数据包四元组

> 每个数据包都要经过pipeline进行流表匹配，所以在pipeline_process_packet函数中我们加一个处理就是获取每个数据包的四元组packet_get_four_tuple，然后进行hash，判断是否进入sketch

```c
/* Extracting the four-tuple from a data packet. */
/**
 * @brief 获取数据包的四元组
 * 
 * @param pkt 数据包元数据
 */
bool 
packet_get_four_tuple(struct packet *pkt){
    struct ofl_match_tlv *f;
    int packet_header;
    uint8_t *packet_val;
    uint8_t flag = 0;
    enum {
        FLAG_IP_SRC = 1,
        FLAG_IP_DST = 2,
        FLAG_TCP_SRC = 4,
        FLAG_TCP_DST = 8,
        FLAG_FOUR_TUPLE = 15
    };
    struct packet_handle_std *handle = pkt->handle_std;

    struct four_tuple *ft = malloc(sizeof(struct four_tuple));

    if (ft == NULL) {
        // 处理内存分配失败的情况
        return false;
    }

    if (!handle->valid){
        //判断数据包里面的match是否有数据，没有就解析并保存到match中
        packet_handle_std_validate(handle);
        if (!handle->valid){
            free(ft);
            return false;
        }
    }
    struct ofl_match *match = &handle->match;
    //循环查找四元组
    HMAP_FOR_EACH(f, struct ofl_match_tlv, hmap_node, &match->match_fields){
        packet_header = f->header;
        packet_val = f->value;
        if(packet_header != 0 && *packet_val != 0){
            switch (packet_header)
            {
            case OXM_OF_IPV4_SRC:
                ft->ip_src = *((uint32_t *)packet_val);
                flag |= FLAG_IP_SRC;
                break;
            case OXM_OF_IPV4_DST:
                ft->ip_dst = *((uint32_t *)packet_val);
                flag |= FLAG_IP_DST;
                break;
            case OXM_OF_TCP_SRC:
                ft->tcp_src = *((uint16_t *)packet_val);
                flag |= FLAG_TCP_SRC;
                break;
            case OXM_OF_TCP_DST:
                ft->tcp_dst = *((uint16_t *)packet_val);
                flag |= FLAG_TCP_DST;
                break;
            default:
                break;
            }
        }
    }
    if(ft != NULL && flag == FLAG_FOUR_TUPLE){
        VLOG_DBG_RL(LOG_MODULE, &rl, "--ft->ip_src: %u, --ft->ip_dst: %u, --ft->tcp_src: %u, --ft->tcp_dst: %u, ", 
                    ft->ip_src, ft->ip_dst, ft->tcp_src, ft->tcp_dst);
        record_sketch(ft);
        free(ft);
        return true;
    }else{
        free(ft);
        return false;
    }
}
```

## 2.计算hash，并记录到sketch中

> 与sketch有关代码，写在packet.c中

```C++
/**
 * @brief 计算hash值
 * 
 * @param ft 数据包四元组
 * @return uint8_t 返回hash值
 */
uint8_t sketch_hash(const struct four_tuple *ft){
    // 对每个成员进行取模操作
    uint32_t ip_src_mod = (ft->ip_src/3) % MOD;
    uint32_t ip_dst_mod = (ft->ip_dst/4) % MOD;
    uint32_t tcp_src_mod = (ft->tcp_src/5) % MOD;
    uint32_t tcp_dst_mod = (ft->tcp_dst/6) % MOD;

    // 求和
    uint32_t sum = ip_src_mod + ip_dst_mod + tcp_src_mod + tcp_dst_mod;

    // 对和再取模
    uint8_t hash_value = sum % MOD;
    return hash_value;
}

/* Compute the hash of a data packet and record it in a sketch. */
/**
 * @brief 计算数据包四元组的hash值并把它记录到sketch中
 * 
 * @param ft 数据包四元组
 */
void record_sketch(const struct four_tuple *ft){
    uint8_t hash_value = sketch_hash(ft);
    
    if(top_k_array[hash_value].exist_flow_flag == 0){
        top_k_array[hash_value].ft.ip_src = ft->ip_src;
        top_k_array[hash_value].ft.ip_dst = ft->ip_dst;
        top_k_array[hash_value].ft.tcp_src = ft->tcp_src;
        top_k_array[hash_value].ft.tcp_dst = ft->tcp_dst;
        top_k_array[hash_value].vote_yes++;
        top_k_array[hash_value].exist_flow_flag = 1;
    }else{
        if(four_tuple_compare(ft, &top_k_array[hash_value].ft) == 0){
            top_k_array[hash_value].vote_yes++;
        }else{
            top_k_array[hash_value].vote_no++;
        }
        if(top_k_array[hash_value].vote_no >= LAMDA * top_k_array[hash_value].vote_yes){
            clear_elephant_node(&top_k_array[hash_value]);
        }
    }  
}
/**
 * @brief 初始化sketch数据结构
 * 
 * @param array 大象流数组
 * @param length 数组长度
 */
void initialize_elephant_array(struct elephant_node *array, uint8_t length){
    for(uint8_t i = 0; i < length; i++){
        clear_elephant_node(&array[i]);
    }
}

/**
 * @brief 清除数组中某一项
 * 
 * @param node 
 */
void clear_elephant_node(struct elephant_node *node){
    node->ft.ip_src = 0;
    node->ft.ip_dst = 0;
    node->ft.tcp_src = 0;
    node->ft.tcp_dst = 0;
    node->vote_no = 0;
    node->vote_yes = 0;
    node->exist_flow_flag = 0;
}

/**
 * @brief 比较两个四元组是否相同
 * 
 * @param a 第一个四元组
 * @param b 第二个四元组
 * @return int 0相同，-1不同
 */
int four_tuple_compare(const struct four_tuple *a, const struct four_tuple *b) {
    if (a->ip_src != b->ip_src) return -1;
    if (a->ip_dst != b->ip_dst) return -1;
    if (a->tcp_src != b->tcp_src) return -1;
    if (a->tcp_dst != b->tcp_dst) return -1;
    return 0; // equal
}
```

## 3.Sketch定期上传数据（构建OpenFlow消息）

> 见[2023-9-23-修改适应ns3的bofuss库源码自定义NewMessage](https://maizhude.github.io/2023/09/23/%E4%BF%AE%E6%94%B9%E9%80%82%E5%BA%94ns3%E7%9A%84bofuss%E5%BA%93%E6%BA%90%E7%A0%81%E8%87%AA%E5%AE%9A%E4%B9%89NewMessage/)，包括构建新OpenFlow消息，交换机发送消息，控制器处理消息，这里只给出在pipeline中获取sketch数据封装成openflow消息并调用dp_send_message发送到控制器，其他步骤在另一篇文章已给出。

```C++
void
pipeline_handle_sketch_data(struct pipeline *pl, const struct sender *sender){
    struct ofl_msg_sketch_data msg;
    msg.header.type = OFPT_SKETCH_DATA;
    if(isInitialFlag == 0){
        initialize_elephant_array(top_k_array, TOP_K);
        isInitialFlag = 1;
    }
    for(uint8_t i = 0; i < TOP_K; i++){
        msg.elephant_flow[i].ip_src = top_k_array[i].ft.ip_src;
        msg.elephant_flow[i].ip_dst = top_k_array[i].ft.ip_dst;
        msg.elephant_flow[i].tcp_src = top_k_array[i].ft.tcp_src;
        msg.elephant_flow[i].tcp_dst = top_k_array[i].ft.tcp_dst; 
        if(top_k_array[i].ft.tcp_src != 0){
            VLOG_DBG_RL(LOG_MODULE, &rl, "pipeline_handle_sketch_data--ft->ip_src: %u, --ft->ip_dst: %u, --ft->tcp_src: %u, --ft->tcp_dst: %u, ", 
                    top_k_array[i].ft.ip_src, top_k_array[i].ft.ip_dst, top_k_array[i].ft.tcp_src, top_k_array[i].ft.tcp_dst);
        }
    }
    initialize_elephant_array(top_k_array, TOP_K);
    dp_send_message (pl->dp, (struct ofl_msg_header *)&msg, sender);
    return;
}
```

