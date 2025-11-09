---
layout:     post
title: ns3.34安装ofswitch13（基于BOFUSS库的openflow13）教程
date:       2023-9-10
author:     MZ
header-img: img/post-bg-kuaidi.jpg
catalog: true
categories:
    - NS3
tags:
    - ns3
    - SDN
---

## 1、安装依赖

```shell
$ sudo apt-get install build-essential gcc g++ python git mercurial unzip cmake pkg-config autoconf libtool libboost-dev
```

<!--more-->

## 2、下载ns3.34

根据官网提示，选择任意一种下载方式下载ns3.34(这里采用方式是Downloading ns-3 Using a Tarball)：

```shell
cd
mkdir tarballs
cd tarballs
wget http://www.nsnam.org/release/ns-allinone-3.34.tar.bz2
tar xjf ns-allinone-3.34.tar.bz2
```

## 3、下载OFSwitch13

```shell
cd /home/your-home/ns-allinone-3.34/ns-3.34/contrib
git clone https://github.com/ljerezchaves/ofswitch13.git
cd /home/your-home/ns-allinone-3.34/ns-3.34/contrib/ofswitch13
git checkout 5.0.1
```

> 注意ns3与ofswitch13之间的版本对应关系，可以在ofswitch13的github仓库里找到：[releases](https://github.com/ljerezchaves/ofswitch13/releases).比如这里的ns3.34对应的是ofswitch13的5.0.1

## 4、下载BOFUSS库（也就是ofsoftswitch13）、编译

```shell
cd /home/your-home/ns-allinone-3.34/ns-3.34/contrib/ofswitch13/lib
git clone https://github.com/ljerezchaves/ofsoftswitch13.git
cd /home/your-home/ns-allinone-3.34/ns-3.34/contrib/ofswitch13/lib/ofsoftswitch13
git checkout v5.0.x
./boot.sh
./configure --enable-ns3-lib
make -j8
```

> 编译完成之后会在lib/ofsoftswitch13/udatapath文件夹下生成一个重要文件 **libns3ofswitch13.a**，ns3编译时就是找是否存在这个文件，有的话才会开启openflow13的编译。

## 5、打补丁

```shell
cd /home/your-home/ns-allinone-3.34/ns-3.34
patch -p1 < contrib/ofswitch13/utils/ofswitch13-src-3_34.patch
patch -p1 < contrib/ofswitch13/utils/ofswitch13-doc-3_34.patch
```

## 6、重新编译ns3

```shell
./waf configure
./waf
```

> 配置之后的正确输出：
>
> ![img]({{ site.url }}/assets/OFSwitch-successful-configuration.png)
>
> ![img]({{ site.url }}/assets/OFSwitch13-enabled.png)

> 编译之后的正确输出：
>
> ![img]({{ site.url }}/assets/OFSwitch-Successful-compile.png)
