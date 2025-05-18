---
layout:     post
title: ubuntu利用docker安装opengauss数据库及测试
subtitle:  ubuntu利用docker安装opengauss数据库及测试
date:       2023-9-30
author:     MZ
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - docker
    - opengauss
    - sql
---

# ubuntu利用docker安装opengauss数据库及测试

## 一、安装

### docker安装

```shell
sudo apt install docker.io
```

### 拉取openguass镜像

```shell
sudo docker pull enmotech/opengauss
```

### 创建容器

```shell
docker run --name opengauss \ 
-p 5432:5432 \ 
-v /home/docker:/var/lib/opengauss/data \
-e GS_NODENAME=gaussdb \
-e GS_USERNAME=gaussdb \
-e GS_PASSWORD=Enmo@123 \
--privileged=true \
--restart=always \
-d enmotech/opengauss:latest 
```

> 密码组合一定是大写字母、小写字母、特殊符号、数字都包含的，密码不对，容器能创建成功，但是没法开启。

### 连接数据库 ,切换到omm用户 ，用gsql连接到数据库

```shell
sudo docker exec -it opengauss bash –进入docker
su - omm --切换为omm用户
gsql  --启动opengauss
```

然后可以输入sql语句建库建表啦。

## 二、建库建表测试

### 1.建立flow_meter库

首先进入到gsql操作界面新建表空间：

```sql
CREATE TABLESPACE tpcds_local RELATIVE LOCATION 'tablespace/tablespace_1' ;
```

然后创建一个新的数据库

```sql
CREATE DATABASE flow_meter WITH TABLESPACE = tpcds_local;
```

可以查看数据库：

```
\l
```

### 2.建表语句

> 因为4张表内容相同，所以这里只定义一张示例表。到时候换一个表名即可。

```sql
CREATE TABLE examples (
            src_ip  VARCHAR(15) NOT NULL,
            dst_ip  VARCHAR(15) NOT NULL,
            src_port INTEGER NOT NULL,  
            dst_port INTEGER NOT NULL,  
            protocol TINYINT NOT NULL,
            timestamp TIMESTAMP,
            flow_duration INTEGER,
            flow_byts_s BINARY_DOUBLE,
            flow_pkts_s BINARY_DOUBLE,
            fwd_pkts_s BINARY_DOUBLE,
            bwd_pkts_s BINARY_DOUBLE,
            tot_fwd_pkts INTEGER,
            tot_bwd_pkts INTEGER,
            totlen_fwd_pkts INTEGER,
            totlen_bwd_pkts INTEGER,
            fwd_pkt_len_max INTEGER,
            fwd_pkt_len_min INTEGER,
            fwd_pkt_len_mean BINARY_DOUBLE,
            fwd_pkt_len_std BINARY_DOUBLE,
            bwd_pkt_len_max INTEGER,
            bwd_pkt_len_min INTEGER,
            bwd_pkt_len_mean BINARY_DOUBLE,
            bwd_pkt_len_std BINARY_DOUBLE,
            pkt_len_max INTEGER,
            pkt_len_min INTEGER,
            pkt_len_mean BINARY_DOUBLE,
            pkt_len_std BINARY_DOUBLE,
            pkt_len_var BINARY_DOUBLE,
            fwd_header_len INTEGER,
            bwd_header_len INTEGER,
            fwd_seg_size_min INTEGER,
            fwd_act_data_pkts INTEGER,
            flow_iat_mean BINARY_DOUBLE,
            flow_iat_max INTEGER,
            flow_iat_min INTEGER,
            flow_iat_std BINARY_DOUBLE,
            fwd_iat_tot INTEGER,
            fwd_iat_max INTEGER,
            fwd_iat_min INTEGER,
            fwd_iat_mean BINARY_DOUBLE,
            fwd_iat_std BINARY_DOUBLE,
            bwd_iat_tot INTEGER,
            bwd_iat_max INTEGER,
            bwd_iat_min INTEGER,
            bwd_iat_mean BINARY_DOUBLE,
            bwd_iat_std BINARY_DOUBLE,
            fwd_psh_flags INTEGER,
            bwd_psh_flags INTEGER,
            fwd_urg_flags INTEGER,
            bwd_urg_flags INTEGER,
            fin_flag_cnt INTEGER,
            syn_flag_cnt INTEGER,
            rst_flag_cnt INTEGER,
            psh_flag_cnt INTEGER,
            ack_flag_cnt INTEGER,
            urg_flag_cnt INTEGER,
            ece_flag_cnt INTEGER,
            down_up_ratio FLOAT4,
            pkt_size_avg FLOAT4,
            init_fwd_win_byts INTEGER,
            init_bwd_win_byts INTEGER,
            active_max INTEGER,
            active_min INTEGER,
            active_mean BINARY_DOUBLE,
            active_std BINARY_DOUBLE,
            idle_max INTEGER,
            idle_min INTEGER,
            idle_mean BINARY_DOUBLE,
            idle_std BINARY_DOUBLE,
            fwd_byts_b_avg BINARY_DOUBLE,
            fwd_pkts_b_avg BINARY_DOUBLE,
            bwd_byts_b_avg BINARY_DOUBLE,
            bwd_pkts_b_avg BINARY_DOUBLE,
            fwd_blk_rate_avg BINARY_DOUBLE,
            bwd_blk_rate_avg BINARY_DOUBLE,
            fwd_seg_size_avg BINARY_DOUBLE,
            bwd_seg_size_avg BINARY_DOUBLE,
            cwe_flag_count INTEGER,
            subflow_fwd_pkts INTEGER,
            subflow_bwd_pkts INTEGER,
            subflow_fwd_byts INTEGER,
            subflow_bwd_byts INTEGER )
```

### 3.插入数据测试

> 这里采用opengauss的copy操作，从csv文件直接复制到数据表当中。因为csv的数据有些问题，所以先提取转换一下。

#### 转换test.csv数据

```python
import csv

# 读取原始 CSV 文件
with open('test.csv', 'r') as f:
    reader = csv.reader(f)
    header = next(reader)  # 读取列标题
    modified_rows = []

    for row in reader:
        # 将包含小数的列（例如第二列）中的小数部分去掉
        modified_row = [row[0], row[1], row[2], row[3], row[4], row[5], int(float(row[6])), 
                        row[7], row[8], row[9], row[10], row[11], row[12], row[13], row[14], 
                        int(float(row[15])), int(float(row[16])), row[17], row[18], 								int(float(row[19])), int(float(row[20])), row[21], row[22], 
                        row[23], row[24], row[25], row[26], row[27], 
                        row[28], row[29], row[30], 
                        row[31], row[32], int(float(row[33])), 
                        int(float(row[34])), row[35], int(float(row[36])), 
                        int(float(row[37])), int(float(row[38])), row[39], 
                        row[40], int(float(row[41])),
                        int(float(row[42])), int(float(row[43])), row[44],
                        row[45], row[46],
                        row[47], row[48], row[49], row[50], row[51], 
                        row[52], row[53], row[54],
                        row[55], row[56], row[57], row[58], row[59], 
                        row[60], int(float(row[61])), int(float(row[62])),
                        row[63], row[64], int(float(row[65])), int(float(row[66])), 
                        row[67], row[68], row[69], row[70],
                        row[71], row[72], row[73], row[74], row[75], 
                        row[76], row[77], row[78],
                        row[79], row[80], row[81]]  # 假设第二列包含小数
        modified_rows.append(modified_row)

# 将修改后的数据保存到新的 CSV 文件
with open('test.csv', 'w', newline='') as f:
    writer = csv.writer(f)
    writer.writerow(header)  # 写入列标题
    writer.writerows(modified_rows)  # 写入修改后的数据
```

#### 执行copy操作

> 把服务器上转换好的csv文件copy到数据库的某个表中。

```sql
COPY unnormal_flow FROM '/home/omm/test.csv' WITH CSV HEADER DELIMITER ','
```

最后查询成功。