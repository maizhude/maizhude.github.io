---
title: tcp_probe文件未找到--换用ftrace进行cwnd测量
date: 2022-11-24 18:07:56
tags:
---

### Error:tcp_probe文件找不到

```shell
rmmod: ERROR: Module tcp_probe is not currently loaded
modprobe: FATAL: Module tcp_probe not found in directory /lib/modules/5.15.0-53-generic

```

网上搜索发现linux内核现在已经不支持tcp_probe了

### 改用ftrace

```shell
  # cd /sys/kernel/debug/tracing
  # echo 1 > events/tcp/tcp_probe/enable
  (run workloads)
  # cat trace
```

