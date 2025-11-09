---
layout:     post
title:     ns3 Tracing系统详解
date:       2022-9-20
author:     MZ
header-img: img/post-bg-2015.jpg
catalog: true
categories:
    - NS3
tags:
    - ns3
---

## 一、为什么要设置Tracing系统

运行ns-3仿真就是为了能够对现实网络结构进行仿真然后得到想要的数据，得到用于研究的输出。先介绍两种不那么灵活的输出方法，一种是打印输出，一种是日志系统，这两种方法对于我们想要得到的精准信息都有些缺陷。打印到标准输出需要进入ns-3核心代码去添加输出语句，当需要的信息变多时，代码将不堪重负；日志系统只能打印某个级别的信息，无法精准的输出自己想要的信息，需要花费大量时间从大量信息中查找那么一两条自己想要的信息实在无法接受。

所以ns-3设置了Tracing系统，为的就是能够精准打印用户想要的信息，还不需要进入核心修改代码。

<!--more-->

## 二、Tracing的简单实现以及三大要素

### 1.三大要素

> 1. tracing sources（跟踪源）
> 2. tracing sinks（跟踪接收器）
> 3. connect（源与接收器的连接）

### 2.简单实现（源与跟踪器在同一文件中/ns-3的fourth.cc）

### a.tracing sources（跟踪源）:

```c++
class MyObject : public Object
{
public:
  static TypeId GetTypeId (void)
  {
    static TypeId tid = TypeId ("MyObject")
      .SetParent (Object::GetTypeId ())
      .SetGroupName ("MyGroup")
      .AddConstructor<MyObject> ()
      .AddTraceSource ("MyInteger",
                       "An integer value to trace.",
                       MakeTraceSourceAccessor (&MyObject::m_myInt),
                       "ns3::TracedValueCallback::Int32")
      ;
    return tid;
  }

  MyObject () {}
  TracedValue<int32_t> m_myInt;
};
```

最重要的一个方法就是AddTraceSource，添加跟踪源。最重要的一个变量是m_myInt。

```c++
/*
*	参数一：此跟踪源的名称
*	参数二：此跟踪源的描述
*	参数三：跟踪的值
*	参数四：TracedValue的类型名称，用于生成文档。
*/
AddTraceSource ("MyInteger",
                       "An integer value to trace.",
                       MakeTraceSourceAccessor (&MyObject::m_myInt),
                       "ns3::TracedValueCallback::Int32")
```

### b.tracing sinks（跟踪接收器）:

```c++
//一般和跟踪源定义在一个文件中，一般在源文件中
void (* traceSink)(int32_t oldValue, int32_t newValue);
//自己实现的跟踪接收器方法
void
IntTrace (int32_t oldValue, int32_t newValue)
{
  std::cout << "Traced " << oldValue << " to " << newValue << std::endl;
}
```

上面两个函数，一个是跟踪接收器的函数签名，一个是跟踪接收器的具体实现，只有帮他俩绑在一起才能实现真正的跟踪功能。

### c.connect（源与接收器的连接）:

```c++
int
main (int argc, char *argv[])
{
  Ptr<MyObject> myObject = CreateObject<MyObject> ();
  myObject->TraceConnectWithoutContext ("MyInteger", MakeCallback(&IntTrace));

  myObject->m_myInt = 1234;
}
```

最后一步，将IntTrace具体实现方法绑定到跟踪接受器上，执行最后一行就会输出：

```
Traced 0 to 1234
```

## 三、改变连接方式--配置路径

上面的源与接收器连接时的TraceConnectWithoutContext方法我们一般很少使用，因为它属于比较底层的代码，我们一般采用Config子系统并且使用配置路径来选择跟踪源。（跟踪源与跟踪接收器不变，只是改变了连接的方法和第一个参数）如：

```c++
std::ostringstream oss;
oss << "/NodeList/"
    << wifiStaNodes.Get (nWifi - 1)->GetId ()
    << "/$ns3::MobilityModel/CourseChange";
//oss.str () = "/NodeList/7/$ns3::MobilityModel/CourseChange"
Config::Connect (oss.str (), MakeCallback (&CourseChange));
```

Config函数会遍历字符串路径，找到最后一个对象属性（MobilityModel），然后相当于创建了一个对象，调用底层TraceConnectWithoutContext或者TraceConnect方法。（下面两行代码就相当于上面的c.connect（源与接收器的连接）的实现）

```c++
Ptr<MobilityModel> mobilityModel = node->GetObject<MobilityModel> ()
    mobilityModel->TraceConnectWithoutContext ("MyInteger", MakeCallback(&IntTrace));
```

> config实际上就是根据路径找到底层的TraceConnect方法，然后调用。所以
>
> ```c++
> //oss.str () = "/NodeList/7/$ns3::MobilityModel/CourseChange"
> Config::Connect (oss.str (), MakeCallback (&CourseChange));
> ```
>
> 这行代码根据路径最终会找到下面的两行代码
>
> ```c++
> Ptr<MobilityModel> mobilityModel = node->GetObject<MobilityModel> ()
>     mobilityModel->TraceConnectWithoutContext ("MyInteger", MakeCallback(&IntTrace));
> ```
>
> 所以采用config方法只是节省了我们自己创建对象的过程。可以更方便我们的使用。

## 四、如何找到ns-3中已存在的跟踪源、配置路径以及回调函数的返回类型和形式参数

### 1.跟踪源

查看API文档侧边栏会有：

- ns-3
  - ns-3 Documentation（ns-3文档）
  - All TraceSources（ 所有跟踪源）
  - All Attributes（所有属性）
  - All GlobalValues（所有全局变量）

![image-20221009100712239]({{ site.url }}/assets/source1.png)

点击All TraceSources会看到ns-3中所有的跟踪源，找到自己想要的那个即可。

### 2.配置路径

在所有跟踪源中找到你想要的之后点击进去，比如MobilityModel点进去如下图，会看到一个描述后面有个More...如下图，点击后会跳到下面。

![image-20221009100302754]({{ site.url }}/assets/config1.png)

跳转之后如图：

![image-20221009100604554]({{ site.url }}/assets/config2.png)

这样就找到了配置路径

### 3.回调函数的返回类型与参数

![image-20221009135000602]({{ site.url }}/assets/signature1.png)

在刚刚的配置路径下面找到回调签名，并点击相关链接跳转，找到

![image-20221009135211981]({{ site.url }}/assets/signature2.png)

```c++
typedef void(* ns3::MobilityModel::TracedCallback) (Ptr< const MobilityModel > model)
```

这行代码里就有需要的形式参数和返回类型，可以明确的说一般的返回类型都是void，所以一般只要关注形参就可以了，这里只有一个形参。

```c++
 .AddTraceSource ("CourseChange", 
                     "The value of the position and/or velocity vector changed",
                     MakeTraceSourceAccessor (&MobilityModel::m_courseChangeTrace),
                     "ns3::MobilityModel::TracedCallback")

TracedCallback<Ptr<const MobilityModel> > m_courseChangeTrace;
```

也可以从源代码中找到添加源的方法，找到对应变量，然后在.h文件中找到变量的定义语句即最后一行代码，从这里我们也可以读出形式参数为<>里的参数，返回类型一般为void上面已经说过了。TracedCallback<>里支持最多八个参数，在底层源码中的模板实现的，具体不展开讲解。

> 至此，完成了简单的查找跟踪源，配置路径，回调函数的形参与返回类型。
