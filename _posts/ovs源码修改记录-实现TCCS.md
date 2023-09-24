---
title: ovs源码修改记录--实现TCCS
date: 2022-12-12 11:56:28
tags: 
---

> 为了实现论文中的思想，需要修改ovs及控制器相关代码以实现扩展openflow协议，写下笔记记录修改过程。

# 拥塞控制信息--OFPT_BUF_CN

## 一、修改./include/openvswitch/ofp-msgs.h

修改该文件下的enum ofpraw和enum ofptype两个枚举

```c++
enum ofpraw{
    //...
    /* OFPT 1.3+ (108): struct ofp13_buf_cn. */
    OFPRAW_OFPT13_BUF_CN,
    /* OFPT 1.3+ (109): struct ofp13_buf_cn. */
    OFPRAW_OFPT13_BUF_CR,
}
//...
enum ofptype{
    //...
    /* TCCS extensions*/
    OFPTYPE_BUF_CN,         /* OFPRAW_OFPT13_BUF_CN. */
    OFPTYPE_BUF_CR,         /* OFPRAW_OFPT13_BUF_CR. */
}
```

<!--more-->

> 注意格式一定要严格，包括注释的格式，因为后面有个文件是根据注释解析的，改为之后进入ovs文件夹内，执行命令：
>
> ```shell
> ./configure --enable-Werror	#开启Werror，展示报错信息
> sudo make #编译
> ```
>
> 下面展示改为可能出现的错误

### 错误一

没有报红色的错误，但是出现了：

```
PYTHONPATH=./python":"$PYTHONPATH PYTHONDONTWRITEBYTECODE=yes /usr/bin/python3 ./build-aux/extract-ofp-msgs \
	./include/openvswitch/ofp-msgs.h lib/ofp-msgs.inc > lib/ofp-msgs.inc.tmp && mv lib/ofp-msgs.inc.tmp lib/ofp-msgs.inc
./include/openvswitch/ofp-msgs.h:466: unexpected syntax between messages
```

这表示改ofp-msgs.h文件时注释或者源代码写错了，因为extract-ofp-msgs这个文件就是解析ofp-msgs.h的，把错误unexpected syntax between messages复制到extract-ofp-msgs找一找，看看哪里出了错。

下面的代码是自己从extract-ofp-msgs挑出来的，用来检验自己注释格式是否正确：

```python
import sys
import os.path
import re
line = "   /* OFPT 1.3+ (30): struct ofp13_buf_cn. */"
comment = line.lstrip()[2:].strip()
print(comment)
comment = comment[:-2].rstrip()
print(comment)
m = re.match(r'([A-Z]+) ([-.+\d]+|<all>) \((\d+)\): ([^.]+)\.$', comment)
print(m)
type_, versions, number, contents = m.groups()
number = int(number)
print(type_, versions, number, contents)
line = "    OFPRAW_OFPT13_BUF_CN,"
m = re.match('\s+(?:OFPRAW_%s)(\d*)_([A-Z0-9_]+),?$' % type_,
                     line)
print(m)
vinfix, name = m.groups()
print(vinfix, name)
rawname = 'OFPRAW_%s%s_%s' % (type_, vinfix, name)
print(rawname)
human_name = '%s_%s' % (type_, name)
print(human_name)
```



## 修改./include/openflow/openflow-1.3.h

### 错误二

出现报红色的错误时，表明ofp-msgs.h文件改的基本差不多了

```
/include/openvswitch/ofp-msgs.h:464:16: error: invalid application of ‘sizeof’ to incomplete type ‘struct ofp13_buf_cn’
  464 |     /* OFPT 1.3+ (108): struct ofp13_buf_cn. */
```

这个错误是没有相关的struct，需要在相关的地方定义，比如这里写的1.3+，那么我就在`./include/openflow/openflow-1.3.h`里修改添加一个struct。

```c++
struct ofp13_buf_cn{
    struct ofp_header           buf_header;     // 8
    struct ofp13_port_stats     buf_port;       // 112
    ovs_be32                    port_buf;       // 4
    ovs_be16                    priority;       // 2
    uint8_t                     pad[2];         // 2
    ovs_be32                    cookie;         // 4
};
//OFP_ASSERT(sizeof(struct ofp13_async_config) == 24);加了这一行报错目前未找到原因
```

![image-20221212120048643](.\ovs源码修改记录-实现TCCS\buf_cn.png)

> 格式按照论文中的进行设置

## 修改./lib/ofp-print.c

### 错误三

```
lib/ofp-print.c: In function ‘ofp_to_string__’:
lib/ofp-print.c:964:5: error: enumeration value ‘OFPTYPE_BUF_CN’ not handled in switch [-Werror=switch]
  964 |     switch (type) {
      |     ^~~~~~
```

很简单添加两个case：

```c++
static enum ofperr
ofp_to_string__(const struct ofp_header *oh,
                const struct ofputil_port_map *port_map,
                const struct ofputil_table_map *table_map, enum ofpraw raw,
                struct ds *string, int verbosity)
{
    if (ofpmsg_is_stat(oh)) {
        ofp_print_stats(string, oh);
    }

    const void *msg = oh;
    enum ofptype type = ofptype_from_ofpraw(raw);
    switch (type) {
            //添加的两个case
		case OFPTYPE_BUF_CN:
        	break;
    	case OFPTYPE_BUF_CR:
        	break;
    }
}
```

## 修改./lib/rconn.c

### 错误四

```
lib/rconn.c: In function ‘is_admitted_msg’:
lib/rconn.c:1345:5: error: enumeration value ‘OFPTYPE_BUF_CN’ not handled in switch [-Werror=switch-enum]
 1345 |     switch (type) {
      |     ^~~~~~
```

很简单添加两个case：

```c++
static bool
is_admitted_msg(const struct ofpbuf *b)
{
    enum ofptype type;
    enum ofperr error;

    error = ofptype_decode(&type, b->data);
    if (error) {
        return false;
    }

    switch (type) {
        //。。。
        case OFPTYPE_BUF_CN://添加
    	case OFPTYPE_BUF_CR://添加
    	default:
        	return true;
    }
    //。。。
}
```

## 修改./lib/ofp-bundle.c
